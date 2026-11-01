# infra/0012 — Workload Placement and Availability Classes

**Status:** Accepted (approved by merging on 2026-11-01)
**Date:** 2026-10-31
**Scope:** Where a workload of the platform runs (virtual machine, LXC container, Kubernetes pod), how in-house software is packaged for it, and which services must be clustered before the platform can call itself a 24/7 DevOps infrastructure. Applies to the on-premise cloud the owner is preparing: several Proxmox nodes hosting the platform, many tenant instances (Odoo among them) and the in-house development targets. Not in scope: the node hardware and cluster mechanics (`infra/0009-hypervisor-cluster`), the database engine choices (`services/0004-database-strategy`), backups (`infra/0008-backup-strategy`).
**Related:** `infra/0001-platform-stack` (Docker first, k3s "when load justifies it"), `infra/0009-hypervisor-cluster`, `infra/0011` resolver pair, `services/0001-opnsense`, `services/0003-netbox-cmdb` (§2 `guest_type`), `services/0004-database-strategy`, `services/0011-container-registry`, `security/0003-hardening`

---

## Context

**Measured on 2026-10-31.** One hypervisor carries the platform: 40 running guests, 94 vCPU allocated on 48 threads, 156 GiB of 377 GiB allocated, 7-day CPU p95 13.6 %, RAM maximum 142 GiB. 36 guests run their service as Docker containers inside an LXC or a VM; `/var/lib/docker` sits on the root filesystem on every one of them. k3s is one VM with the system pods, a demo, Headlamp and the Portainer agent, storage class `local-path`: nothing in production runs as a pod. The second hypervisor (different CPU vendor, 1 GbE uplinks only) hosts three stopped Windows guests.

**What is clustered today:** PostgreSQL (Patroni, three members, etcd three), Vault (Raft, three), Redis (two members, three Sentinels), Authentik (two), the resolver (two behind one VRRP address, `infra/0011`).

**What still stops everything when it stops:** the ingress (one Traefik), the database endpoint (one Pgpool-II, every consumer connects through it), the Vault address (every client names node 1; the Raft quorum does not help when node 1 is down; Shamir seal means a manual unseal after any restart), the firewall (one OPNsense VM), the hypervisor itself. Three full outages in one week came from these: a host reboot (2026-10-24, three days), a firewall panic (2026-10-30, 2.5 minutes), a planned RAM upgrade.

**Owner direction, 2026-10-31**, after the capacity review: pods "as much as possible, not all"; in-house development ships "Docker AND pod compliant" by default; several Proxmox nodes become the on-premise cloud hosting many Odoo instances and the in-house targets; "all that can be clustered" among the central services is clustered, GitLab included; the firewall becomes a pair on static addresses (five static addresses are available on the default ISP, three in use today by the legacy firewall, the production OPNsense and the test OPNsense; the second ISP is PPPoE/DHCP and can be made static for a monthly fee; a third ISP at 10 Gb/s symmetric arrives within weeks).

`infra/0001` only says "start with Docker, switch to k3s when load justifies it". No ADR says what runs where, what "clustered" means per service, or in which order the single points close. This ADR does.

## Decision

### 1. Placement matrix

| Class | Examples | Runs as | Why |
|---|---|---|---|
| Network and platform appliances | firewall, backup server, resolver pair, VPN coordinator and relay, mail (vendor: no Kubernetes support), Windows guests | **VM** | own kernel, live migration, vendor packaging |
| Stateful core | PostgreSQL members, etcd, Redis + Sentinel, Vault, object storage, step-ca | **LXC** (today) → **pods only through the vendor's Kubernetes path** (CloudNativePG is already the revision trigger of `services/0004`) | the vendor's documented HA already runs; a move must not lower it |
| Stateless platform services | ingress, identity, registry, package proxies, CMDB, admin tools, dashboards, wiki, collaboration tools | Docker in LXC today → **pod** once Phase 2 exists | self-healing, rolling updates, replicas |
| Source control | GitLab | **VM(s)**, vendor reference architecture (§5) | the vendor's Kubernetes repository storage is beta (GitLab 19.1) |
| Tenant workloads (on-premise cloud) | Odoo instances, in-house applications, their test tiers | **pod**, one namespace per tenant | scheduling, quotas, isolation, density |
| CI runners, builds | GitLab runner, Kaniko | **pod** (Kubernetes executor) | ephemeral by nature |

A workload that does not fit a row is placed by a one-line ADR amendment, never by habit.

### 2. Packaging rule for in-house software

Every in-house service ships, from the same repository and the same CI pipeline: one container image (pushed to the platform registry, pinned by digest, `security/0003`), one Compose file for the developer's machine, one Helm chart for the platform. The chart is the deployment artefact; Compose never reaches a platform host. A service that cannot be run from its chart is not releasable.

### 3. Availability classes

| Class | Meaning | Rule |
|---|---|---|
| **A — central** | the platform stops without it | clustered with the vendor's documented HA mechanism; no single instance; reached through an address that survives one member (VRRP address, DNS set, or the vendor's own endpoint); failover tested and recorded |
| **B — workload** | one service or tenant stops | pod with ≥ 2 replicas when stateless; otherwise single instance with a tested restore and a recorded RTO |
| **C — tooling** | nobody is blocked | single instance, backup and restore test per `infra/0008` |

Class A services and their state on 2026-10-31:

| Service | Vendor HA mechanism | Today | Gap |
|---|---|---|---|
| Hypervisor | Proxmox cluster, HA groups, replicated or shared storage | one node | `infra/0009` Phase 0 below |
| Firewall | OPNsense CARP + pfsync + config sync (active/passive; no active/active exists) | one VM | §4 |
| Resolver | two instances, VRRP address | done | consumers move to the address (PR 2) |
| Ingress | Traefik ≥ 2 behind a VRRP address, or the k3s ingress with ≥ 2 replicas | one LXC | pair + address (Phase 1) |
| Identity | Authentik server ≥ 2, worker ≥ 2 | two members behind the single ingress | ingress gap |
| Secrets | Vault Raft 3 | three members, clients name node 1, Shamir seal | clients → address of the three; unseal procedure documented and rehearsed (auto-unseal needs a second trust root: decide in the security pass) |
| Database | Patroni 3 + etcd 3; endpoint Pgpool-II watchdog ≥ 2 with a delegate address | cluster done, endpoint single | endpoint pair (Phase 1) |
| Cache | Redis + 3 Sentinels | done | — |
| Registry | Harbor HA (external PostgreSQL/Redis/object storage, ≥ 2 stateless replicas) | one LXC | Phase 3 as pods |
| Object storage | SeaweedFS ≥ 3 masters + ≥ 2 volume servers, or Ceph RGW with the cluster | one LXC | storage pass (`services/0009`) |
| Source control | GitLab reference architecture with HA (§5) | one LXC | Phase 2+ |
| Monitoring and logging | Prometheus pair or Thanos, Loki replicas, Alertmanager cluster ≥ 2 | one LXC | Phase 3 |
| Backup server | PBS: one instance per site is the vendor model; redundancy is the second datastore and the off-site copy | done | — |

### 4. Firewall pair

CARP uses one address per node plus one shared virtual address per WAN subnet, all in the parent interface's subnet, synchronised over a dedicated pfsync link, with outbound NAT translated to the virtual address (vendor how-to). Consequences:

- **Default ISP (static block of five):** three addresses for the production pair (node, node, virtual), one for the test firewall, one spare. The legacy firewall's address is released when its last flow is migrated.
- **Second ISP (PPPoE/DHCP):** CARP cannot own a PPP session. Either the static option with at least three addresses, or the operator's CPE in router mode with the pair in a private transfer VLAN behind it. The choice is made when the fee is known; until then this WAN stays on the master only and the backup node re-dials on failover (vendor: "Disconnect dialup interfaces").
- **Third ISP (10 Gb/s):** ordered with a static block of at least three addresses or a CPE in router mode; both firewall nodes need a 10 GbE port on different hypervisors, and the switch trunks every ISP VLAN to every node (`infra/0009`). Throughput of a firewall VM at 10 Gb/s with IDS is measured before it is promised.
- The second firewall node runs on a different hypervisor, or the pair protects nothing.

### 5. GitLab

GitLab's smallest architecture with HA is the 3,000-user reference architecture, and the vendor documents the reductions allowed for smaller installations: lower node specs, Rails and Sidekiq on two shared nodes, PostgreSQL and PgBouncer at two nodes each, Consul, Redis Sentinel and Praefect at an odd number of at least three, an external PostgreSQL and a standalone-with-HA Redis instead of GitLab's own. Gitaly Cluster on Kubernetes is beta in GitLab 19.1 and de-prioritised by the vendor in favour of a new storage backend, so GitLab stays on VMs.

Decision: GitLab is class A and is built as the 3k architecture with the documented reductions, on the hypervisor cluster, reusing the platform's PostgreSQL and Redis and the platform's object storage. Until the cluster exists, GitLab runs as one instance with a tested restore and a recorded RTO (class B rule); its repository storage choice (Praefect or the vendor's successor) is taken at Phase 2 against the then-current vendor documentation.

### 6. Order of work

| Phase | Done when |
|---|---|
| **0 — Hypervisor cluster** (`infra/0009`) | the current node, the second node and the third node form one cluster with 10 GbE on every node, replicated or shared storage, HA groups; a guest survives the loss of its node by restart on another |
| **1 — Close the single points on the current runtime** | ingress pair + address, database endpoint pair + address, Vault clients reach all three members, firewall CARP pair across two nodes; each failover tested and recorded in the service's ADR |
| **2 — Kubernetes** | three k3s server VMs on three nodes, storage through a CSI driver on the cluster storage, ingress and secrets integration, backup through CSI snapshots or Velero (`infra/0008` chapter) |
| **3 — Stateless services to pods** | one service per PR through the compliance walk, images unchanged, roles replaced by charts |
| **4 — Stateful only through the vendor path** | CloudNativePG or equivalent, decided per `services/0004` revision trigger |
| **5 — On-premise cloud** | namespace, quota, NetworkPolicy and PostgreSQL per tenant; tenant onboarding and offboarding as one playbook; tenants recorded in the CMDB (`services/0003`) |

Nothing moves to a pod before Phase 2; on one hypervisor a pod dies with the node exactly like an LXC does.

## Consequences

- `infra/0001`'s "switch to k3s when load justifies it" is replaced by this matrix and order.
- The service register gains two columns, mirrored as NetBox custom fields (`services/0003` §2): `runtime` (vm | lxc | pod) and `availability_class` (A | B | C). The compliance walk checks both.
- Terraform modules gain a second data volume for Docker hosts (CIS Docker Benchmark 1.1.1) and the k3s server VM set; Ansible gains the Traefik pair, the Pgpool-II watchdog pair, the Vault client address and the CARP pair in the firewall catalog.
- Every in-house repository gains a Helm chart and a chart lint in CI.
- More guests of class A (two to three times for ingress, endpoint, GitLab) — within the measured headroom of one node and trivial on three.

## Alternatives considered

- **Everything as pods now.** Rejected: on one hypervisor nothing is gained, every role is rewritten before the cluster exists, and the firewall, backup server, mail and Windows guests cannot be pods.
- **Stay Docker-in-LXC only.** Rejected: owner direction; a multi-tenant cloud of many application instances needs a scheduler, quotas and namespaces, not one LXC per tenant.
- **GitLab Helm chart.** Rejected for now: repository storage on Kubernetes is beta and de-prioritised by the vendor.
- **Active/active firewalls.** Not offered by the vendor; CARP is active/passive.

## CISO mapping

| Framework | Controls covered |
|---|---|
| ISO 27001:2022 | A.8.14 (redundancy of information processing facilities — class A), A.5.29 / A.5.30 (continuity, ICT readiness — order of work, tested failovers), A.8.6 (capacity — measured headroom per phase), A.8.20 / A.8.22 (network security, segregation — firewall pair, tenant namespaces and policies), A.8.31 (separation of development, test and production — packaging rule), A.8.9 (configuration management — placement decided by ADR, recorded in the CMDB) |
| NIS2 | Art. 21(2)(c) business continuity; Art. 21(2)(e) security in acquisition, development and maintenance (packaging rule) |

## Revision triggers

- `infra/0009` is accepted with a different node count or storage design.
- GitLab ships its Raft-based repository storage as generally available.
- A vendor named in §1 publishes or withdraws Kubernetes support (Mailcow, OPNsense, PBS).
- The second or third ISP contract fixes the addressing model.
- The first tenant workload is onboarded (Phase 5 details become their own ADR).
