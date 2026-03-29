# VM Template Specification

**Status:** Draft v4 — review 2026-03-29  
**Last updated:** 2026-03-29  
**Scope:** All VMs and LXCs provisioned via Terraform on Proxmox PoC  
**Reviewer:** @yboujraf

Each property has a numeric ID. Legend: ✅ confirmed | ⚠️ open decision | ❌ blocked | 🔄 updated

---

## 0. Proxmox Storage — Current State (verified via API)

| Storage ID | Type | Backend | Content | Mount |
|---|---|---|---|---|
| `poc-data` | **ZFS pool** | `tank/poc-data` (local ZFS) | VM images + rootdir (LXC) | `/tank/poc-data` |
| `local-lvm` | LVM-thin | `pve/data` VG | VM images + rootdir | N/A |
| `poc-iso` | NFS | NAS `10.6.224.6:/volume1/srv-proxmox-poc-01-iso` | ISO + templates | `/mnt/pve/poc-iso` |
| `poc-backup` | NFS | NAS `10.6.224.6:/volume1/srv-proxmox-poc-01-backup` | Backups only | `/mnt/pve/poc-backup` |
| `local` | dir | `/var/lib/vz` | ISO, snippets, templates | N/A |

**Decision confirmed:** `poc-data` = ZFS pool. VMs run on ZFS. NFS is ISO library + backup storage only. This is correct and does not change.

---

## 1. OS Support

| ID | Property | Value | Status |
|---|---|---|---|
| 1.1 | Linux | ✅ Primary | ✅ |
| 1.2 | Templates | **Debian 12**, **Ubuntu 24.04 LTS**, **Rocky Linux 9** | ✅ |
| 1.3 | Windows | Deferred Phase 4+ | ✅ |

---

## 2. Identity & Locale

| ID | Property | Value | Status |
|---|---|---|---|
| 2.1 | Hostname pattern | `vm-<service>-<env>-<nn>` | ✅ |
| 2.2 | Locale | `fr_BE.UTF-8` | ⚠️ re-verify |
| 2.3 | Timezone | `Europe/Brussels` | ✅ |
| 2.4 | Keyboard | `be` | ✅ |
| 2.5 | Shell | `/bin/bash` | ✅ |
| 2.6 | Default user | `by-systems` | ✅ |
| 2.7 | Root SSH | Disabled (Ansible) | ✅ |
| 2.8 | Search domain | `by-systems.arpa` | ✅ |
| 2.9 | DNS | Per VLAN/switch (§6.8) — driven by NetBox | ✅ |

---

## 3. Compute

| ID | Property | Value | Status |
|---|---|---|---|
| 3.1 | CPU type | **`x86-64-v2-AES`** — hardware-agnostic, live-migratable, supports AES-NI | ✅ confirmed |
| 3.2 | CPU sockets | `1` | ✅ |
| 3.3 | CPU cores | Variable per role (§10) | ✅ |
| 3.4 | RAM | Variable per role (§10) | ✅ |
| 3.5 | Ballooning | Disabled | ✅ |
| 3.6 | NUMA | Disabled | ✅ |

> **3.1:** `x86-64-v2-AES` is the correct permanent choice. Agnostic — works across any modern x86 CPU. Any CPU in the Proxmox list that is x86-64-v2 compatible can be assigned. Enables live migration between nodes regardless of exact CPU model.

---

## 4. Storage

| ID | Property | Value | Status |
|---|---|---|---|
| 4.1 | SCSI controller | `virtio-scsi-single` | ✅ |
| 4.2 | iothread | Enabled | ✅ |
| 4.3 | VM / LXC storage | **`poc-data` (ZFS pool `tank/poc-data`)** — all VMs and LXCs run here | ✅ verified |
| 4.4 | Disk format on ZFS | `raw` — best performance, native ZFS snapshots | ✅ |
| 4.5 | ISO / template storage | `poc-iso` (NFS from NAS) — ISOs and cloud-init templates only | ✅ |
| 4.6 | Backup storage | `poc-backup` (NFS from NAS) — Proxmox backups only | ✅ |
| 4.7 | NFS role | **Backup + ISO only. Never VM/LXC disks.** NFS is not a VM runtime backend | ✅ clarified |
| 4.8 | Boot disk size | Variable per role (§10) | ✅ |
| 4.9 | Discard / TRIM | Enabled | ✅ |
| 4.10 | Backup | On demand **or** triggered by NetBox VM tag `backup:enabled` | ✅ |
| 4.11 | Snapshot | On demand — manual or triggered pre-change by CI pipeline. No automated schedule in PoC | ✅ |

> **4.3 — Confirmed from Proxmox API:** `poc-data` is a ZFS pool (`type: zfspool`, pool: `tank/poc-data`). VMs run on ZFS. `local-lvm` exists as fallback but is not the default. NFS (`poc-iso`, `poc-backup`) is storage for ISOs and backups exclusively.

---

## 5. Network

| ID | Property | Value | Status |
|---|---|---|---|
| 5.1 | Interface count | Depends on VM role — see matrix below | 🔄 |
| 5.2 | OOB interface | Only for break-glass VMs — not default | 🔄 |
| 5.3 | VLAN strategy | Agnostic — VLAN IDs from NetBox, not hardcoded in template | 🔄 |
| 5.4 | Bridge naming | Follows NetBox VLAN naming — see §7.3 | 🔄 |
| 5.5 | IP assignment | From NetBox IPAM — Terraform reads NetBox API | ⚠️ Phase 2 |
| 5.6 | Gateway | From NetBox VLAN prefix | ⚠️ Phase 2 |
| 5.7 | MTU | Must match Arista switch config per VLAN — 1500 default, 9000 storage | ⚠️ Arista dependency |
| 5.8 | SDN | Proxmox SDN for intra-hypervisor VM isolation + OPNsense for L3 routing/FW/internet | 🔄 |
| 5.9 | IPv6 | Dual-stack by default | ✅ |
| 5.10 | Network model | `virtio` | ✅ |
| 5.11 | PCIe passthrough | Per-VM explicit — WAN NICs to OPNsense, PTP NIC to GM VM, GPU | ✅ |

> **5.1 — Interface matrix per role:**
>
> | VM role | Interfaces | Notes |
> |---|---|---|
> | Standard infra / app VM | 1 × service VLAN | Normal case |
> | Critical VM needing break-glass | 1 × service VLAN + 1 × OOB | Only when VLAN failure = locked out |
> | OPNsense FW VM | WAN1 + WAN2 (passthrough) + LAN VLANs | Full appliance |
> | VoIP VM | 1 × service VLAN + 1 × VoIP VLAN | Separate bridge for QoS |
> | K3s node | 1 × cluster VLAN | |
> | Supervision / metrics VM | 1 × MGMT-equivalent bridge for fabric access | See §7.3 |

> **5.3 — VLAN agnostic strategy:**
> Template does not hardcode VLAN IDs. VLAN assignment is a variable passed by Terraform, which reads from NetBox. VLAN IDs are defined in the network topology session (D.6) and stored in NetBox as the single source. This keeps the template portable across PoC, staging, and production environments.

> **5.8 — SDN + OPNsense model:**
> - **Inside hypervisor:** Proxmox SDN handles VM-to-VM traffic on same host — no external path needed, low latency, policy enforced at hypervisor level
> - **Between VMs needing L3 routing or internet:** traffic goes through OPNsense (routing + FW + NAT)
> - **OPNsense** = all firewall rules as code (`ansibleguy.opnsense` Ansible collection), full REST API, no manual rules
> - **SDN isolation rules:** VMs in SDN zone can be isolated from each other at hypervisor level — additional layer on top of OPNsense

---

## 6. Cloud-init (Linux)

| ID | Property | Value | Status |
|---|---|---|---|
| 6.1 | User | `by-systems` | ✅ |
| 6.2 | Password | `ci_password` Terraform var | ✅ |
| 6.3 | SSH keys | See note — per-VM keypair, pub key injected at provision | 🔄 |
| 6.4 | Upgrade on boot | `false` | ✅ |
| 6.5 | Packages on boot | `false` | ✅ |
| 6.6 | Locale | `fr_BE.UTF-8` — needs re-verification | ⚠️ |
| 6.7 | Timezone | `Europe/Brussels` | ✅ |
| 6.8 | DNS | Per VLAN/switch topology (see note) | 🔄 |
| 6.9 | Search domain | `by-systems.arpa` | ✅ |
| 6.10 | Cloud-init drive | `ide2` | ✅ |

> **6.3 — SSH key strategy (confirmed):**
> Each VM has its own unique `by-systems` keypair, named by VM hostname: `by-systems@vm-netbox-poc-01`.
> - Private key → stored in Vault at `secret/ssh/<hostname>/by-systems`
> - Public key → injected via cloud-init at provision time
> - Ansible retrieves private key from Vault before connecting
> - No shared master key across VMs
>
> The public key injected comes from **Vault** — Terraform reads `vault_generic_secret.ssh_<hostname>.public_key` and passes it to cloud-init. Keys are generated once per VM and never rotate unless explicitly triggered.

> **6.8 — DNS per VLAN/switch (clarified):**
>
> | Context | DNS resolver | Reason |
> |---|---|---|
> | MGMT VLAN on Arista 7060 (has VRF) | VRF interface IP on switch | Isolated fabric — resolves fabric devices only |
> | Arista 7048 (no VRF) | **NOT on MGMT VLAN by default** — only for DR/short test exception | Hardware limitation — no VRF segmentation possible. Exception only, not standard |
> | Service VLANs (infra, app, etc.) | OPNsense VLAN interface IP | Resolves `by-systems.arpa` + upstream via DoT/DoH |
> | External DNS | Never direct — always through OPNsense DoT/DoH enforcement | FW rule blocks direct port 53 outbound |
>
> OPNsense handles this cleanly — rule-based DNS forwarding per interface/VLAN, DoT/DoH enforced, no manual config needed (Ansible-driven).

---

## 7. Security Baseline (Terraform layer)

| ID | Property | Value | Status |
|---|---|---|---|
| 7.1 | QEMU guest agent | Enabled | ✅ |
| 7.2 | Serial port | None by default — add explicitly for appliances needing serial console | ✅ |
| 7.3 | Network bridges | See note — naming follows NetBox VLAN, no hardcoded vmbrMGMT confusion | 🔄 |
| 7.4 | VGA | `std` headless / `virtio` desktop | ✅ |
| 7.5 | Protection flag | **PoC: off. Production stateful VMs: on.** Delete only through NetBox (CMDB cleanup + neo4j) | 🔄 |
| 7.6 | Start on boot | Enabled | ✅ |
| 7.7 | Boot order | `scsi0` | ✅ |

> **7.3 — Bridge naming and role clarification:**
>
> Previous confusion: `vmbrMGMT` was used for an application — that was wrong. Bridges are named by function, not by application.
>
> | Bridge | VLAN role | Traffic |
> |---|---|---|
> | `vmbrOOB` | OOB (break-glass) | Human emergency access only |
> | `vmbrFAB` | Fabric / MGMT | Supervision, orchestration, Ansible, metrics collection from fabric devices (Arista, NAS, Proxmox) — VRF-aware on 7060 |
> | `vmbrAPP` | Application | Service VMs — routed through OPNsense |
> | `vmbrSTOR` | Storage | ZFS replication, NFS, future Ceph — MTU 9000, Arista dependency |
> | `vmbrVOIP` | VoIP | Separate bridge for QoS/DSCP marking |
> | `vmbrWAN1` | WAN ISP1 | OPNsense only |
> | `vmbrWAN2` | WAN ISP2 | OPNsense only |
>
> **Naming is a placeholder — final names follow NetBox VLAN names once topology is defined (D.6).**
>
> **vmbrFAB (supervision/orchestration bridge):**
> Ansible, Terraform, metrics collectors (Prometheus node_exporter, SNMP, etc.) use this bridge to reach fabric devices. On Arista 7060 this maps to the VRF interface — provides full fabric reachability. On Arista 7048 (no VRF) it reaches only devices on that switch's accessible VLANs.
>
> **PCIe passthrough for OPNsense WAN:**
> Two options:
> - **vmbrWAN1/2 (virtual bridge):** OPNsense gets virtual NIC. Proxmox handles physical NIC. If VM dies, bridge stays up but traffic stops. Simpler, less performance overhead.
> - **PCIe passthrough (dedicated NIC):** OPNsense owns the physical NIC directly. Maximum performance, no hypervisor overhead. If VM dies, NIC is unavailable until VM restarts. No vmbr dependency.
>
> **Recommendation for FW/WAN:** PCIe passthrough for OPNsense WAN NICs. vmbr for LAN-side VLANs (more flexible, can trunk multiple VLANs on one virtual NIC).
>
> **PTP NIC passthrough (GM VM):** Intel NIC with hardware timestamping assigned directly to a VM designated as Grandmaster. This is the correct model — PTP needs direct hardware access for sub-microsecond accuracy.

---

## 8. Security Hardening (Ansible layer)

Baseline: sync with `odoo-install` SSH hardening. Extend from there.

| ID | Property | Value | Role |
|---|---|---|---|
| 8.1 | SSH port | `22222` | `role-ssh-hardening` |
| 8.2 | SSH key-only | Yes | `role-ssh-hardening` |
| 8.3 | Root SSH | Disabled | `role-ssh-hardening` |
| 8.4 | Ciphers | From odoo-install baseline | `role-ssh-hardening` |
| 8.5 | ufw | Default deny | `role-ufw` |
| 8.6 | fail2ban | SSH + per-service | `role-fail2ban` |
| 8.7 | NTP | **Chrony** → fabric NTP source | `role-chrony` |
| 8.8 | auditd | Enabled | `role-auditd` |
| 8.9 | Sudo | `by-systems` passwordless | `role-base` |
| 8.10 | MOTD | Disabled — no OS/version banner | `role-base` |
| 8.11 | unattended-upgrades | Security only | `role-unattended-upgrades` |
| 8.12 | sysctl | Network + FS hardening | `role-sysctl` |

---

## 9. Ansible Compatibility

| ID | Property | Value | Status |
|---|---|---|---|
| 9.1 | Python3 | In all 3 cloud images | ✅ |
| 9.2 | SSH port | `22222` post-hardening | ✅ |
| 9.3 | Bootstrap sequence | Port 22 → hardening → port 22222 | ✅ |
| 9.4 | Privilege escalation | `become: yes` + sudo | ✅ |
| 9.5 | Inventory Phase 1 | Static `hosts.yml` | ✅ |
| 9.6 | Inventory Phase 2+ | NetBox dynamic inventory | ⚠️ |
| 9.7 | Connection | `ssh` (Linux), `winrm` deferred (Windows) | ✅ |
| 9.9 | OS-specific roles | Role vars per OS family | ⚠️ |
| 9.10 | Synology DSM | Ansible collection `by_systems.dsm` wrapping `lib-synology-dsm` — CI-tested | 🔄 |

> **9.10:** Build as full Ansible collection from the start — not just a role. Collection structure enables CI loop testing (molecule + GitHub Actions). Low risk: lib is validated, API is stable. Target: `ansible_collections/by_systems/dsm/` in `ansible-platform` repo.

---

## 10. Role-based Sizing Defaults

**Deployment model:** All services run as **Docker containers** on their respective VMs (docker-compose or single container), except where noted. No bare-metal `apt install` for application services. OPNsense is an image-based appliance. GitLab uses bundled nginx internally but **Traefik** handles external TLS termination and routing.

| ID | Role | CPU | RAM (MB) | Disk (GB) | Storage | Deployment |
|---|---|---|---|---|---|---|
| 10.1 | Bootstrap / utility | 1 | 1024 | 10 | poc-data | N/A |
| 10.2 | NetBox | 2 | 4096 | 30 | poc-data | Docker |
| 10.3 | Vault | 2 | 2048 | 20 | poc-data | Docker |
| 10.4 | Authentik | 2 | 4096 | 20 | poc-data | Docker |
| 10.5 | GitLab CE | 4 | 8192 | 50 | poc-data + NFS (repos/artifacts) | Docker — local disk for app, NAS NFS **and/or S3** for repo storage + artifacts |
| 10.6 | GitLab Runner | 2 | 4096 | 50 | poc-data | Docker |
| 10.7 | Nexus OSS | 2 | 4096 | 50 | poc-data + NFS/S3 | Docker — artifacts on NAS/S3 |
| 10.8 | OPNsense | 2 | 2048 | 10 | poc-data | **Appliance image** — not Docker |
| 10.9 | PostgreSQL (shared) | 2 | 4096 | 50 | poc-data | Docker |
| 10.10 | Redis (shared) | 1 | 2048 | 10 | poc-data | Docker |
| 10.11 | **Traefik** | 1 | 1024 | 10 | poc-data | Docker — TLS termination, routing for all services |
| 10.12 | K3s / K8s node | 4 | 8192 | 80 | poc-data | On 2nd Proxmox node |
| 10.13 | Monitoring (Grafana/Loki/Prometheus) | 2 | 4096 | 50 | poc-data + NAS for long-term metrics | Docker |

> **10.5 — GitLab storage split:**
> - VM local disk (ZFS `poc-data`): GitLab application, DB (PoC bundled), config
> - NAS NFS share: Git repo data (`/var/opt/gitlab/git-data`) — large, grows unbounded
> - S3 (future or MinIO on NAS): CI artifacts, container registry, LFS objects
> - This split keeps the VM disk predictable and offloads bulk storage to NAS/S3
>
> **10.11 — Traefik:**
> Single Traefik instance per environment handles:
> - TLS termination (Let's Encrypt or internal CA via Vault PKI)
> - Routing to all Docker services by hostname
> - GitLab uses internal nginx, but Traefik sits in front for external TLS — GitLab nginx handles internal GitLab-to-GitLab communication only
> - No Apache anywhere

---

## 11. NetBox as Source of Truth

NetBox is the orchestrator. It owns: IP allocation, VLAN definitions, VM records, naming convention, DNS per prefix, environment labels, backup flags.

| ID | What NetBox owns | Drives |
|---|---|---|
| 11.1 | IPAM (IP per VM) | Terraform cloud-init IP, Ansible inventory, DNS A/AAAA records |
| 11.2 | VLAN definitions | Terraform NIC config, Proxmox bridge assignment |
| 11.3 | Naming convention | VM hostname, DNS record, Ansible host_vars |
| 11.4 | DNS server per prefix | Cloud-init nameserver config |
| 11.5 | VM role/tags | Ansible group assignment, Terraform template selection |
| 11.6 | Backup flag | Proxmox backup job trigger |
| 11.7 | Environment | Ansible inventory group, Terraform workspace |
| 11.8 | Physical devices | Arista/NAS/Proxmox targeting for Ansible |

> **Bridge naming follows NetBox VLAN names (D.11 confirmed).** When topology is defined in NetBox, bridge names are generated from VLAN slugs. No manual naming.

> **VM protection (7.5):** Deletion of a protected VM requires:
> 1. NetBox record updated (status → decommissioned)
> 2. CMDB cleanup (neo4j if used)
> 3. Only then Terraform destroy allowed
> This prevents accidental deletion of stateful VMs outside of change management.

---

## 12. Open Decisions

| ID | Question | Status |
|---|---|---|
| D.1 | CPU type | ✅ `x86-64-v2-AES` confirmed |
| D.2 | Storage | ✅ ZFS `poc-data` for VMs, NFS for ISO + backup only |
| D.3 | UPS on Synology | ✅ confirmed — writeback cache safe |
| D.4 | SSH port | ✅ Ansible day-1 only, Terraform does not manage post-provision security |
| D.5 | Windows | ✅ Deferred Phase 4 |
| D.6 | Network topology | ⚠️ Dedicated session — VLAN IDs, IP ranges, dual-stack, ISP config from 3COM |
| D.7 | SSH keys | ✅ Per-VM keys, Vault-stored, `by-systems` user |
| D.8 | PostgreSQL HA | Single VM PoC → cluster Phase 3 |
| D.9 | OPNsense | ✅ Confirmed |
| D.10 | Proxmox SDN | ✅ Enable alongside OPNsense |
| D.11 | Bridge naming | ✅ Follows NetBox VLAN names |

---

## 13. Storage Summary

| Backend | Type | VM disks | LXC rootdir | ISO | Backup | Notes |
|---|---|---|---|---|---|---|
| `poc-data` | ZFS (`tank/poc-data`) | ✅ Default | ✅ | ❌ | ❌ | All VMs run here |
| `local-lvm` | LVM-thin | Fallback only | Fallback | ❌ | ❌ | Not default |
| `poc-iso` | NFS (NAS) | ❌ | ❌ | ✅ | ❌ | ISO + cloud templates |
| `poc-backup` | NFS (NAS) | ❌ | ❌ | ❌ | ✅ | Proxmox backup jobs |
| `local` | dir | ❌ | ❌ | ✅ | ✅ | Legacy/fallback |
