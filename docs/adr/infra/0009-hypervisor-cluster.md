# infra/0009 — Hypervisor Cluster (compute platform)

**Status:** Draft — proposal for approval (hardware refresh decision)
**Date:** 2026-09-23
**Deciders:** @yboujraf
**Scope:** The physical compute platform: node class, cluster topology, storage tiers, the networks the cluster itself uses, the growth path from one node to three, and how the Kubernetes layer sits on top. Does not define per-zone network policy (`infra/0004`), backup policy (`infra/0008`) or any service.
**Related:** `infra/0001-platform-stack` (Proxmox VE + k3s), `infra/0004-network-architecture` (zones; "Proxmox clustering supported by design"; Storage zone reserved for Ceph), `infra/0008-backup-strategy`, `naming/0001-infra` (node names), `services/0001-opnsense-provisioning` (the firewall is a VM of this cluster), `services/0004-database-strategy` (PostgreSQL HA)

---

## Context

The platform runs on one physical node from 2012 (one socket, 6 cores / 12 threads, 2.5 GHz, 188 GiB RAM, seven rotating SAS disks under ZFS). Thirty guests are allocated 76 vCPU on those 12 threads. Seven days of the node's own accounting (2026-09-22) measured the platform at **3.9 cores average, 7.7 cores at the 95th percentile, 63 GiB RAM at the 95th percentile, 91 GiB of disk in use**. Capacity is not the constraint; two other things are:

- **Per-thread speed and disk latency.** The services every other service depends on (identity provider, shared PostgreSQL, the firewall's PPPoE and IDS) run on one or two threads each and wait on rotating disks. Incidents of 2026-09-22 (connection storms, 5xx waves) were latency effects, not saturation.
- **No host resilience.** One node is a single point of failure for every service, including the firewall and the backup server that runs on it.

A hardware refresh is on the table (vendor build sheet of 2026-09), and the platform direction is "every service containerised for a later Kubernetes move" (`infra/0001`). Several decisions cannot be changed after the first node is installed (cluster identity, node names, corosync addresses, VLAN trunking, disk layout). This ADR fixes them before the first node is bought.

## Decision

### 1. Topology — a cluster from day one, three nodes with Ceph as the target

- The compute platform is a **Proxmox VE cluster**. The first node is installed as *node one of the cluster* (cluster created on day one), never as a standalone host; nodes join later without reinstall.
- **One node:** guest storage on local ZFS mirrors; backups and the quorum device live on a separate small box (§4).
- **Two nodes:** ZFS replication between the nodes plus the external quorum device; HA for the guests that support it.
- **Three nodes (target):** **Ceph** on the nodes' NVMe drives, three replicas, `min_size 2`; the cluster survives the loss of one node with quorum intact. Guests move from ZFS to Ceph disk by disk (live for VMs, short restart for containers).
- Up to three nodes the Ceph public and cluster networks run on a **full mesh of direct 25 GbE links** (two ports per node); a 25 GbE switch is only required at node four or for external 25 GbE consumers.

### 2. Node class (a hardware class, not a vendor)

| Item | Requirement | Reason |
|---|---|---|
| Form / CPU | 1U, single socket, **≥ 32 cores / 64 threads at ≥ 3.5 GHz base** | per-thread speed first; the platform peak is under 8 cores, cores beyond 32 buy nothing today |
| RAM | ≥ 384 GB ECC | 6× the measured peak; room for workstations and the Kubernetes layer |
| Boot | 2 × 480 GB M.2 mirror | operating system only |
| Data | **4 × 3.84 TB NVMe U.2**, bought with the node | ZFS mirrors on one node, Ceph OSDs at three; never a RAID controller in front of them (HBA / pass-through mode) |
| Network | 2 × 25 GbE SFP28 (storage and cluster mesh) + 4 × 1 GbE (management, corosync ring 0 and 1, OOB) | corosync wants its own low-latency link; storage traffic never shares it |
| Power / BMC | dual PSU on two feeds; BMC on the OOB zone (`infra/0004` zone 1) | |
| Reference builds | the "Small A" class (single-socket EPYC 9355P) meets every row; the equivalent Dell 1U chassis meets it **when ordered with the same CPU** and its RAID controller in HBA mode | the 128- to 192-core builds of the same sheet are rejected: lower clock, twice the power, cores the platform cannot use |
| 100 GbE (optional) | a dual-port NVIDIA ConnectX is optional for Ceph rebuild bandwidth | **mandatory only for media hosts built on the NVIDIA Rivermax stack, which runs on NVIDIA ConnectX / BlueField adapters only. Those hosts run the vendor's own bare-metal Ubuntu image and are never virtualised, so they are never nodes of this cluster** (§6) |

### 3. Networks the cluster uses (zones per `infra/0004`)

- **Corosync:** dedicated 1 GbE link pair (ring 0 and ring 1), never on the storage or guest networks.
- **Ceph public and cluster:** the 25 GbE mesh, in the Storage zone.
- **Guest networks:** every node port that carries guests is an **identical trunk** of all platform VLANs **and both ISP VLANs**, so the firewall VM, and later its CARP pair, can run on any node. Bridge and SDN zone names are identical on every node (`infra/0004 §Proxmox SDN`). The ISP handoffs therefore terminate on the switch, not on a node port.
- **Management and OOB:** per `infra/0004`; the BMCs and the switch console stay in the OOB zone.

### 4. Storage tiers

| Tier | One node | Three nodes |
|---|---|---|
| Boot | ZFS mirror (M.2) | same |
| Guest data (VM and container volumes) | ZFS mirrors on NVMe | Ceph RBD, three replicas |
| Object storage (S3 for backups, evidence, registries) | the existing S3 service | Ceph RGW may replace it — decided in the storage service pass, not here |
| Backups | **outside the cluster**: the backup server on the small separate box, NAS and off-site per `infra/0008` | same |

### 5. Growth path and the cutover from the current node

1. Switch: identical trunks on the node ports, ISP VLANs included.
2. Node one installed as cluster node one, ZFS on its NVMe, subscription taken.
3. The current node **joins the cluster**; VMs migrate live (the platform already pins a CPU type both generations share), containers move with a short restart; the firewall moves inside a maintenance window using the proven rebuild or a migration once its ISP VLANs are on the trunk; the old node leaves the cluster.
4. Node two: ZFS replication + quorum device. Node three: Ceph, guests move to it, ZFS data pools retired.
5. Every step is applied by Ansible (`pve_host` role) from the identity catalog (node names, corosync and storage addresses); nothing is configured by hand.

### 6. The Kubernetes layer

Proxmox with Ceph is the substrate for what is not container-shaped: the firewall pair, the backup server, jump host, customer workstations. The Kubernetes cluster (k3s or RKE2, `infra/0001`) runs as three control-plane VMs spread over the three nodes plus worker VMs; its persistent volumes come from the same Ceph through the CSI driver (block for databases, object for what the S3 service does today). Services move to it as they are containerised; the hardware does not change. Real-time media workloads (§2, Rivermax class) run bare-metal on the vendor's Ubuntu image on their own certified hosts, never as guests or nodes of this cluster; the same node class may be bought for them (spares, support), with the ConnectX added, and they enter the platform as a **vendor-appliance** service class: BMC on the OOB zone, media VLANs per `infra/0004`, administration through the bastion, configuration backup, monitoring only as the vendor permits, no platform hardening beyond what the vendor supports.

### 7. Resilience targets enabled by three nodes (each is a service pass, tracked separately)

Firewall pair (CARP), PostgreSQL replica (`services/0004`), ingress pair, identity-provider pair, three-node Kubernetes control plane.

### 8. Operations

- One Proxmox subscription per socket (community tier suffices) for the enterprise repository and a support contract as audit evidence; vendor hardware support with next-business-day terms recorded per node.
- Node names per `naming/0001-infra`; addresses and VLAN ids in the identity catalog; the node's hardware inventory in NetBox (`infra/0004`: physical wiring is operational state).

## Consequences

**Enables:** host resilience, live migration for maintenance, a real Kubernetes cluster, workstations at scale, retirement of the 2012 node.
**Constrains:** node one must be bought with its NVMe and 25 GbE ports even though a single node does not need the mesh; the switch must trunk the ISP VLANs before the firewall can leave the old node; hostnames and corosync addresses are final once joined.
**Known risks:** two-node interim depends on the external quorum device; Ceph at three nodes gives 15 TB usable from 46 TB raw (three replicas), which is ample today but sets the growth unit to "one node with four drives".

## Revision triggers

- A fourth node, or external 25 GbE consumers → the 25 GbE switch.
- Ceph RGW adopted → the object-storage row of §4 and `infra/0008` destinations.
- A second site → `infra/0004 §Revision triggers` (second cluster, same zone names).

## CISO mapping

| Framework | Controls covered |
|---|---|
| ISO 27001:2022 | A.8.14 (redundancy of information processing facilities — three nodes, Ceph), A.8.6 (capacity management — measured 95th percentile drives the node class), A.5.30 (ICT readiness for business continuity — live migration, external backup box), A.7.8 (equipment siting — dual feeds, BMC on OOB) |
| NIS2 | Art. 21(2)(c) (business continuity — host loss survivable), Art. 21(2)(d) (supply chain security — vendor support terms per node), Art. 21(2)(i) (asset management — nodes in NetBox) |
| GDPR | N/A — no direct control mapping |
