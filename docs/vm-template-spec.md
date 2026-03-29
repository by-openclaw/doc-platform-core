# VM Template Specification

**Status:** Draft v3 — review 2026-03-28/29  
**Last updated:** 2026-03-29  
**Scope:** All VMs and LXCs provisioned via Terraform on Proxmox PoC  
**Reviewer:** @yboujraf  

Each property has a numeric ID. Legend: ✅ confirmed | ⚠️ open decision | ❌ blocked | 🔄 updated

---

## 1. OS Support

| ID | Property | Value | Status |
|---|---|---|---|
| 1.1 | Linux supported | ✅ Primary target | ✅ |
| 1.2 | Linux OS templates | **Debian 12**, **Ubuntu 24.04 LTS**, **Rocky Linux 9** — 3 separate base images | ✅ |
| 1.3 | Windows | Deferred Phase 4+ (developer workstations only) | ✅ deferred |

> **Why 3 templates:** Debian 12 = infra default. Ubuntu 24.04 = workloads needing Ubuntu packages. Rocky 9 = RHEL-compatible / enterprise software. Each is a separate Proxmox cloud-init image.

---

## 2. Identity & Locale

| ID | Property | Value | Status |
|---|---|---|---|
| 2.1 | Hostname pattern | `vm-<service>-<env>-<nn>` (e.g. `vm-netbox-poc-01`) | ✅ |
| 2.2 | Locale | `fr_BE.UTF-8` | ✅ needs re-verification |
| 2.3 | Timezone | `Europe/Brussels` | ✅ |
| 2.4 | Keyboard | `be` | ✅ |
| 2.5 | Default shell | `/bin/bash` | ✅ |
| 2.6 | Default user | `by-systems` — single user, no split | ✅ |
| 2.7 | Root SSH login | Disabled (Ansible hardening) | ✅ |
| 2.8 | Search domain | `by-systems.arpa` | ✅ |
| 2.9 | DNS | Per VLAN assignment from NetBox — not hardcoded (see §6.8) | 🔄 |

---

## 3. Compute

| ID | Property | Value | Status |
|---|---|---|---|
| 3.1 | CPU type | `host` (PoC) → `x86-64-v2-AES` before cluster | ⚠️ switch before clustering |
| 3.2 | CPU sockets | `1` (2nd socket deferred) | ✅ |
| 3.3 | CPU cores | Variable per role (§10) | ✅ |
| 3.4 | RAM | Variable per role (§10) | ✅ |
| 3.5 | Memory ballooning | Disabled | ✅ |
| 3.6 | NUMA | Disabled | ✅ |

---

## 4. Storage

| ID | Property | Value | Status |
|---|---|---|---|
| 4.1 | SCSI controller | `virtio-scsi-single` | ✅ |
| 4.2 | iothread | Enabled | ✅ |
| 4.3 | Storage backend | See note below — ZFS preferred when available | 🔄 clarified |
| 4.4 | Disk format on NFS | `qcow2` | ✅ |
| 4.5 | Disk format on ZFS/LVM | `raw` | ✅ |
| 4.6 | Boot disk size | Variable per role (§10) | ✅ |
| 4.7 | Disk cache (NFS) | `writeback` — see note | 🔄 clarified |
| 4.8 | Discard / TRIM | Enabled | ✅ |
| 4.9 | SSD emulation | Enabled | ✅ |
| 4.10 | Backup | **On demand or driven by NetBox VM tag** `backup:scheduled` | 🔄 |
| 4.11 | Snapshot | **On demand** — no automated snapshots in PoC. Triggered manually or by CI pipeline pre-change | 🔄 |

> **4.3 — Storage backend decision:**
>
> | Backend | Where VMs run | Format | Use case |
> |---|---|---|---|
> | `local-zfs` | ZFS pool on Proxmox node local disk | `raw` | **Preferred** — native snapshots, compression, checksums, no single point of failure from NAS |
> | `local-lvm` | LVM-thin pool on Proxmox node | `raw` | Fallback if no ZFS configured |
> | `poc-data` (Synology NFS) | NAS over network | `qcow2` | Shared storage for cluster live migration — not for OS disk normally |
>
> **Recommendation:** Configure ZFS on the Proxmox PoC node for local VM storage. Use Synology NFS (`poc-data`) only for:
> - Shared ISO/template library
> - Large data disks (GitLab repos, Nexus artifacts)
> - Cluster live migration (VM must be on shared storage for that feature)
>
> If PoC node has no ZFS pool yet → action item before tomorrow.

> **4.7 — Disk cache writeback clarification:**
> `writeback` = Proxmox tells the VM "write is done" as soon as data hits the NAS RAM cache — before it hits disk. Faster writes. Risk: if NAS loses power before flushing cache → data loss. **Acceptable only if Synology has UPS.** If no UPS → use `none` (safer, slower). Default to `none` until UPS confirmed.

---

## 5. Network

| ID | Property | Value | Status |
|---|---|---|---|
| 5.1 | Interfaces per VM | **Depends on role** — see note | 🔄 |
| 5.2 | OOB interface | Only for VMs requiring break-glass access independent of VLAN state | 🔄 |
| 5.3 | VLAN per environment | Yes — separate VLAN per env (dev/staging/prod) is the standard | 🔄 |
| 5.4 | Additional interfaces | Appliances (OPNsense, routers) get explicit per-NIC config — not templated | ✅ |
| 5.5 | IP assignment | From NetBox — NetBox is source of truth for IP allocation | 🔄 |
| 5.6 | Gateway | From NetBox VLAN prefix config | 🔄 |
| 5.7 | MTU | Must match Arista switch config on same VLAN — 1500 default, 9000 for storage VLAN | ⚠️ Arista dependency |
| 5.8 | Firewall / SDN | OPNsense handles routing/FW. Proxmox SDN handles VLAN provisioning programmatically. See note | 🔄 |
| 5.9 | IPv6 | Dual-stack by default | ✅ |
| 5.10 | Network model | `virtio` | ✅ |
| 5.11 | PCIe / IO passthrough | Per-VM explicit config — IOMMU, GPU, PTP NIC, WAN NICs | ✅ |

> **5.1/5.2 — Interface strategy per role:**
>
> | VM role | Interfaces | Reason |
> |---|---|---|
> | Standard infra VM | 1 × service VLAN | Simpler — OPNsense handles routing between VLANs |
> | VM needing break-glass OOB | 1 × service VLAN + 1 × OOB VLAN | Only for critical VMs where VLAN failure = locked out |
> | OPNsense firewall VM | 1 × WAN ISP1 (PCIe passthrough or dedicated vmbr) + 1 × WAN ISP2 + 1 × per service VLAN | Full network appliance |
> | K3s / workload node | 1 × cluster VLAN | |
>
> **Principle:** Don't add OOB to every VM. Add it only where losing the service VLAN means losing management access. NetBox tracks which VMs have OOB and which don't.

> **5.3 — Environment segmentation:**
> Standard approach: separate VLAN per environment tier. This aligns with ISO 27001 network segmentation requirements.
> ```
> VLAN 10 — MGMT       (Arista mgmt, Proxmox, NAS, OOB)
> VLAN 20 — INFRA      (NetBox, Vault, Authentik, GitLab)
> VLAN 30 — APP-POC    (PoC application workloads)
> VLAN 40 — APP-PROD   (Production workloads — future)
> VLAN 50 — STORAGE    (Ceph/NFS traffic — MTU 9000)
> VLAN 60 — K8S        (Kubernetes cluster internal)
> VLAN 100 — WAN-ISP1  (OPNsense WAN 1)
> VLAN 101 — WAN-ISP2  (OPNsense WAN 2)
> ```
> Exact IDs TBD in network topology session. This is a starting point.

> **5.8 — SDN + OPNsense clarification:**
> - **OPNsense** = Layer 3 routing + firewall + NAT + VPN. All inter-VLAN routing goes through OPNsense. Full REST API, Ansible collection (`ansibleguy.opnsense`). Rules are not manual — they are code.
> - **Proxmox SDN** = Layer 2 VLAN provisioning inside Proxmox. Creates virtual bridges and VLAN segments for VMs without touching physical switch config. Complementary to OPNsense, not a replacement.
> - **Arista switches** = Physical fabric. Access/trunk VLANs, routing between switches, potentially SDN integration via eAPI.
> - These three layers work together. Nothing is manual.

---

## 6. Cloud-init (Linux only)

| ID | Property | Value | Status |
|---|---|---|---|
| 6.1 | User | `by-systems` | ✅ |
| 6.2 | Password | `ci_password` Terraform variable | ✅ |
| 6.3 | SSH keys | See note — one key approach, Vault-stored | 🔄 |
| 6.4 | Upgrade packages on boot | `false` | ✅ |
| 6.5 | Install packages on boot | `false` | ✅ |
| 6.6 | Locale | `fr_BE.UTF-8` — needs re-verification | ⚠️ |
| 6.7 | Timezone | `Europe/Brussels` | ✅ |
| 6.8 | DNS | Per VLAN / switch topology — see note | 🔄 |
| 6.9 | Search domain | `by-systems.arpa` | ✅ |
| 6.10 | Cloud-init drive | `ide2` | ✅ |

> **6.3 — SSH key strategy (KISS):**
>
> Single user `by-systems`. No split between Ansible user / Terraform user / human user. One user, one set of keys, SSO for future human access.
>
> Key generation approach — matching `odoo-install` pattern:
> - **One unique SSH keypair per VM** generated at provision time
> - Key named by VM: `by-systems@vm-netbox-poc-01`
> - Private key stored in **Vault** under `secret/ssh/<hostname>/by-systems`
> - Public key injected via cloud-init
> - Ansible retrieves key from Vault before connecting
>
> This means: no shared "master key" across all VMs. Compromise of one VM does not give access to others. Clean, auditable, follows Vault best practices.
>
> **Open decision D.7:** Confirm this approach or use a shared key (simpler, lower security). Recommendation: per-VM keys from the start.

> **6.8 — DNS per VLAN (not global):**
>
> DNS resolver depends on which VLAN the VM is on AND which switches are in that fabric:
>
> | VLAN / context | DNS resolver | Reason |
> |---|---|---|
> | MGMT VLAN (Arista fabric) | VRF IP on the switch serving that VLAN | Isolated fabric — only MGMT reachable. Must resolve fabric devices locally |
> | Arista 7048 (no VRF) | Use MGMT VLAN IP directly — no VRF isolation possible on this switch | Hardware limitation |
> | Service VLANs (infra, app) | OPNsense VLAN interface IP | OPNsense resolves `by-systems.arpa` + upstream via DoT/DoH |
> | External DNS | Never direct — always via OPNsense DoT/DoH | Enforced at firewall, no bypass |
>
> NetBox stores the DNS server per prefix/VLAN. Cloud-init gets it from Terraform which reads NetBox. This is not hardcoded.

---

## 7. Security Baseline (Terraform layer)

| ID | Property | Value | Status |
|---|---|---|---|
| 7.1 | QEMU guest agent | Enabled | ✅ |
| 7.2 | Serial port | None by default — add explicitly for headless appliances needing serial console | ✅ |
| 7.3 | PCIe / IO passthrough | Per-VM explicit only — WAN NICs to OPNsense, GPU/PTP to specific VMs (see note) | 🔄 |
| 7.4 | VGA type | `std` headless infra / `virtio` desktop VMs | ✅ |
| 7.5 | Protection flag | See note | 🔄 |
| 7.6 | Start on boot | Enabled | ✅ |
| 7.7 | Boot order | `scsi0` only | ✅ |

> **7.3 — Network bridge naming convention and OPNsense passthrough:**
>
> Today Proxmox has `vmbrOOB` and `vmbrMGMT`. Proposed naming convention:
>
> | Bridge name | VLAN | Used for |
> |---|---|---|
> | `vmbrOOB` | OOB (untagged, PoC) | Break-glass VM access |
> | `vmbrMGMT` | MGMT VLAN | Proxmox host management |
> | `vmbrAPPS` | APP VLAN | Application VM traffic |
> | `vmbrSTOR` | STORAGE VLAN | NFS/Ceph traffic (MTU 9000) |
> | `vmbrWAN1` | WAN ISP1 | OPNsense WAN interface 1 |
> | `vmbrWAN2` | WAN ISP2 | OPNsense WAN interface 2 |
>
> OPNsense VM gets:
> - `vmbrWAN1` → physical NIC or PCIe passthrough for ISP1
> - `vmbrWAN2` → physical NIC or PCIe passthrough for ISP2
> - `vmbrAPPS` (or per-VLAN bridge) → LAN side, inter-VLAN routing
>
> **PCIe passthrough** (IOMMU required on Proxmox host): used when OPNsense needs direct access to physical NIC for WAN, or when a VM needs a PTP NIC, GPU, or any other PCIe device. Configured per-VM in Terraform, not in base template.

> **7.5 — Protection flag:**
> Proxmox VM protection flag (`protection: true`) prevents:
> - Accidental VM deletion via API or UI
> - Disk deletion when VM is deleted
>
> **PoC:** off (need to be able to recreate quickly during testing)
> **Production:** on for all stateful VMs (databases, Vault, GitLab)
> Set via Terraform variable — `var.protected = true/false` per VM role.

---

## 8. Security Hardening (Ansible layer)

Baseline: sync with `odoo-install` SSH hardening role. Extend from there.

| ID | Property | Value | Ansible role |
|---|---|---|---|
| 8.1 | SSH port | `22222` | `role-ssh-hardening` |
| 8.2 | SSH key-only auth | Yes | `role-ssh-hardening` |
| 8.3 | Root SSH | Disabled | `role-ssh-hardening` |
| 8.4 | Allowed ciphers | From odoo-install baseline | `role-ssh-hardening` |
| 8.5 | ufw | Enabled, default deny | `role-ufw` |
| 8.6 | fail2ban | SSH + per-service | `role-fail2ban` |
| 8.7 | NTP | **Chrony** → MGMT VLAN NTP source | `role-chrony` |
| 8.8 | auditd | Enabled | `role-auditd` |
| 8.9 | Sudo | `by-systems` passwordless | `role-base` |
| 8.10 | MOTD | Disabled — no OS/version banner leak | `role-base` |
| 8.11 | Unattended-upgrades | Security only | `role-unattended-upgrades` |
| 8.12 | sysctl hardening | Network + filesystem hardening | `role-sysctl` |

---

## 9. Ansible Compatibility

| ID | Property | Value | Status |
|---|---|---|---|
| 9.1 | Python3 | Included in all 3 cloud images | ✅ |
| 9.2 | SSH port in inventory | `ansible_port: 22222` after hardening | ✅ |
| 9.3 | Bootstrap sequence | Port 22 → hardening → port 22222 (see below) | ✅ |
| 9.4 | Privilege escalation | `become: yes` + sudo passwordless | ✅ |
| 9.5 | Inventory Phase 1 | Static `hosts.yml` | ✅ |
| 9.6 | Inventory Phase 2+ | NetBox dynamic inventory | ⚠️ blocked on NetBox |
| 9.7 | Connection Linux | `ssh` | ✅ |
| 9.8 | Connection Windows | `winrm` — deferred Phase 4 | ⚠️ |
| 9.9 | OS-specific roles | Role vars per OS family | ⚠️ |
| 9.10 | Synology DSM Ansible | Build Ansible module wrapping `lib-synology-dsm` — see note | 🔄 |

> **9.3 Bootstrap sequence:**
> ```
> Day 0: Terraform → VM provisioned (SSH port 22, key from cloud-init)
> Day 1: Ansible bootstrap play (inventory group: bootstrap, port 22)
>   → role-base (user, sudo, hostname)
>   → role-ssh-hardening (port → 22222, key-only)
>   → role-chrony, role-ufw, role-fail2ban
> Day 2+: All plays use port 22222 (inventory group: hardened)
> ```

> **9.10 — Ansible DSM module:**
> `lib-synology-dsm` is a Python lib. Two integration options:
>
> | Option | Effort | Risk | Value |
> |---|---|---|---|
> | Ansible `uri` module calling DSM API directly | Low | Low | Works but verbose playbooks |
> | Ansible role wrapping `lib-synology-dsm` Python calls | Medium | Low | Clean, reusable |
> | Full Ansible collection (`ansible_collections.by_systems.dsm`) | High | Low | Best long-term |
>
> **Recommendation:** Start with Ansible role wrapping lib calls. Build full collection when GitLab CI is live. No risk — lib is already validated.

---

## 10. Role-based Sizing Defaults

| ID | Role | CPU | RAM (MB) | Disk (GB) | Storage | Notes |
|---|---|---|---|---|---|---|
| 10.1 | Utility / bootstrap | 1 | 1024 | 10 | ZFS local | Temporary |
| 10.2 | NetBox | 2 | 4096 | 30 | ZFS local | Uses shared PostgreSQL + Redis |
| 10.3 | Vault | 2 | 2048 | 20 | ZFS local | |
| 10.4 | Authentik | 2 | 4096 | 20 | ZFS local | Uses shared PostgreSQL + Redis |
| 10.5 | GitLab CE | 4 | 8192 | 100 | NAS NFS | Bundled Postgres PoC, external prod |
| 10.6 | GitLab Runner | 2 | 4096 | 50 | ZFS local | |
| 10.7 | Nexus OSS | 2 | 4096 | 100 | NAS NFS | Artifact storage |
| 10.8 | OPNsense | 2 | 2048 | 10 | ZFS local | Full API + Ansible |
| 10.9 | PostgreSQL (shared) | 2 | 4096 | 50 | ZFS local | Shared by NetBox, Authentik |
| 10.10 | Redis (shared) | 1 | 2048 | 10 | ZFS local | Shared cache |
| 10.11 | K3s / K8s node | 4 | 8192 | 80 | ZFS local | See note |
| 10.12 | Monitoring | 2 | 4096 | 50 | NAS NFS | Grafana/Loki/Prometheus |

> **10.11 — K3s placement:**
> K3s node runs as a VM on the **2nd Proxmox node** (the one being set up tomorrow). Not on the PoC node that already hosts NetBox, Vault, etc. Reasons:
> - Separates workload plane from control/infra plane
> - 2-node Proxmox cluster: node 1 = infra VMs, node 2 = K3s + future workloads
> - If second node is an existing server: fine as a single K3s node to start, expand to multi-node K3s later
> - K3s on a single VM = valid for PoC. Production K3s = 3 nodes minimum (control plane quorum)

---

## 11. NetBox as Source of Truth

NetBox is the orchestrator — it holds the authoritative data that drives Terraform, Ansible, and DNS.

| ID | What NetBox owns | Terraform reads | Ansible reads | Status |
|---|---|---|---|---|
| 11.1 | IP address allocation (IPAM) | VM IP via NetBox API | Inventory IP | ⚠️ blocked on NetBox |
| 11.2 | VLAN definitions | VLAN ID for VM NIC | Network config | ⚠️ |
| 11.3 | VM naming convention | Hostname | `ansible_hostname` | ⚠️ |
| 11.4 | DNS server per prefix | DNS in cloud-init | DNS in role vars | ⚠️ |
| 11.5 | VM role / tags | Template selection | Role assignment | ⚠️ |
| 11.6 | Backup flag | `backup: true/false` | N/A | ⚠️ |
| 11.7 | Environment label | `env: poc/prod` tag | Inventory group | ⚠️ |
| 11.8 | Physical device inventory | N/A | Arista/NAS targeting | ⚠️ |

> **11 — NetBox flow (Phase 2+):**
> ```
> Engineer defines in NetBox:
>   → Add prefix/VLAN
>   → Allocate IP
>   → Create VM record (name, role, env, tags)
>
> Terraform:
>   → Reads NetBox VM record via API
>   → Provisions Proxmox VM with correct IP, VLAN, storage, sizing
>   → Tags Proxmox VM with NetBox ID for cross-reference
>
> Ansible:
>   → NetBox dynamic inventory plugin pulls VM list
>   → Groups VMs by role/env/OS tags
>   → Applies correct roles and config per group
>
> DNS (Bind/OPNsense):
>   → Reads NetBox IPAM
>   → Auto-generates A/AAAA records for all VMs
> ```
>
> **Phase 1 (now):** Static `hosts.yml` + manual NetBox-like discipline (naming, IPs tracked in a spreadsheet or RAID.md until NetBox is live).

---

## 12. Open Decisions

| ID | Question | Decision |
|---|---|---|
| D.1 | CPU type | `host` now → `x86-64-v2-AES` before cluster |
| D.2 | Storage: ZFS vs LVM-thin vs NAS | ZFS on Proxmox node (action: verify ZFS pool exists). NAS for shared/large disks |
| D.3 | UPS on Synology | Must confirm before enabling writeback cache |
| D.4 | SSH port change | Ansible day-1 only. Terraform does not manage post-provision security |
| D.5 | Windows | Deferred Phase 4 |
| D.6 | Network topology | Dedicated session needed — VLAN IDs, IP ranges, dual-stack, VRF strategy |
| D.7 | SSH key strategy | Per-VM keys in Vault vs shared key — recommendation: per-VM |
| D.8 | PostgreSQL HA | Single VM PoC → 3-VM cluster Phase 3 |
| D.9 | OPNsense | ✅ confirmed — replaces pfSense |
| D.10 | Proxmox SDN | Enable alongside OPNsense for VLAN provisioning via API |
| D.11 | vmbr naming convention | Proposed in §7.3 — confirm before cluster setup |

---

## 13. Storage: NAS vs Ceph

| | Synology NAS (ZFS-backed NFS) | Local ZFS on Proxmox | Ceph |
|---|---|---|---|
| Min nodes | 1 | 1 | 3 |
| 2-node cluster | ✅ live migration | ✅ local only (no migration) | ⚠️ no quorum |
| Snapshots | ✅ (qcow2 on NFS) | ✅ native ZFS | ✅ native |
| Performance | Network-bound | Best (local NVMe/SSD) | Near-local |
| HA | ✅ with NAS | ❌ | ✅ |
| Phase 1–2 | ✅ NAS for shared, ZFS for local | ✅ | ❌ |
| Phase 3+ | Valid | Valid | ✅ preferred |
| Prerequisite | NAS + Proxmox NFS config | ZFS pool on node | 3 nodes + storage VLAN on Arista + OSD disks |
