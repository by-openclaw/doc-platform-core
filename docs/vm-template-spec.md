# VM Template Specification

**Status:** Draft v2 — incorporating @yboujraf review 2026-03-28  
**Last updated:** 2026-03-28  
**Scope:** All VMs and LXCs provisioned via Terraform on Proxmox PoC  
**Reviewer:** @yboujraf  

Each property has a numeric ID for direct reference in review comments.  
Legend: ✅ confirmed | ⚠️ open decision | ❌ blocked | 🔄 updated from review

---

## 1. OS Support

| ID | Property | Value | Status |
|---|---|---|---|
| 1.1 | Linux supported | ✅ Primary target | ✅ |
| 1.2 | Linux OS templates | **Debian 12**, **Ubuntu 24.04 LTS**, **Rocky Linux 9** | 🔄 3 templates required |
| 1.3 | Windows supported | Deferred Phase 4+ (developer workstations only) | ✅ deferred |
| 1.4 | Windows management | WinRM + Sysprep — separate template family entirely | ✅ deferred |
| 1.5 | Windows cloud-init | Not supported — different bootstrap mechanism | ✅ noted |

> **Why 3 Linux templates:**
> - **Debian 12** — platform infra default (NetBox, Vault, Authentik, GitLab)
> - **Ubuntu 24.04 LTS** — workloads requiring Ubuntu-specific packages or Snap
> - **Rocky Linux 9** — RHEL-compatible workloads, enterprise software (e.g. OPNsense VM, future SAP/Oracle constraints)
>
> Each template is a separate Proxmox base image (cloud-init enabled). Terraform references the template ID by name via variable.

---

## 2. Identity & Locale

| ID | Property | Value | Status |
|---|---|---|---|
| 2.1 | Hostname pattern | `vm-<service>-<env>-<nn>` (e.g. `vm-netbox-poc-01`) | ✅ |
| 2.2 | Locale | `fr_BE.UTF-8` | ✅ validated (needs re-verification — see 6.6) |
| 2.3 | Timezone | `Europe/Brussels` | ✅ validated |
| 2.4 | Keyboard layout | `be` | ✅ validated |
| 2.5 | Default shell | `/bin/bash` | ✅ |
| 2.6 | Default user | `by-systems` only — **one user, clean** | 🔄 rune key renamed (see 6.3) |
| 2.7 | Root login | Disabled via SSH | ⚠️ Ansible hardening role |
| 2.8 | Search domain | `by-systems.arpa` | ✅ confirmed |
| 2.9 | DNS resolver | VLAN GW / VRF IP (see §6.8) — **no 1.1.1.1 if DoT/DoH enforced** | 🔄 updated |

---

## 3. Compute

| ID | Property | Value | Status |
|---|---|---|---|
| 3.1 | CPU type | `host` (PoC) → `x86-64-v2-AES` (cluster) | 🔄 see note |
| 3.2 | CPU sockets | `1` (second socket deferred — future expansion) | 🔄 confirmed |
| 3.3 | CPU cores | Variable per role (§10) | ✅ |
| 3.4 | Memory (RAM) | Variable per role (§10) | ✅ |
| 3.5 | Memory ballooning | Disabled | ✅ |
| 3.6 | NUMA | Disabled (single socket) | ✅ |
| 3.7 | CPU hotplug | Disabled | ✅ |
| 3.8 | Memory hotplug | Disabled | ✅ |

> **3.1 CPU type clarification:**
>
> | | `host` | `x86-64-v2-AES` |
> |---|---|---|
> | Performance | Max — passes all host CPU flags | ~5–10% lower |
> | Live migration | ❌ Only works if both nodes have **identical** CPU model | ✅ Works across different CPUs |
> | Risk | Split-brain if nodes differ: VM will refuse to start on the other node | Safe |
> | PoC (single node) | ✅ Fine | ✅ Fine |
> | Cluster (2+ nodes) | ⚠️ Only safe if CPU model identical — **verify before cluster** | ✅ Always safe |
>
> **Decision: keep `host` for now. Switch to `x86-64-v2-AES` before forming cluster unless both Proxmox nodes have confirmed identical CPU model, core count, and thread count.**

---

## 4. Storage

| ID | Property | Value | Status |
|---|---|---|---|
| 4.1 | SCSI controller | `virtio-scsi-single` | ✅ |
| 4.2 | iothread | Enabled | ✅ |
| 4.3 | Default storage backend | **`poc-data` (Synology NFS)** — not local-lvm | 🔄 updated |
| 4.4 | Disk format on poc-data | `qcow2` (required for NFS-backed storage + snapshots) | 🔄 |
| 4.5 | local-lvm / LVM-thin use | Only for OS-only VMs needing max IOPS (future — not PoC) | 🔄 |
| 4.6 | Boot disk size | Variable per role (§10) | ✅ |
| 4.7 | Disk cache (NFS) | `writeback` | ⚠️ confirm with NAS team |
| 4.8 | Discard / TRIM | Enabled | ✅ |
| 4.9 | SSD emulation | Enabled | ✅ |
| 4.10 | Backup | Enabled for all — schedule TBD | ⚠️ |
| 4.11 | Snapshot policy | Manual (PoC) | ⚠️ |

> **poc-data:** Synology NAS shared folder exposed via NFS to Proxmox. All PoC VMs live here. Local-lvm is available as fallback for high-IOPS cases but not the default.

---

## 5. Network

| ID | Property | Value | Status |
|---|---|---|---|
| 5.1 | Interface strategy | **2 interfaces minimum:** OOB (break-glass) + production VLAN | 🔄 updated |
| 5.2 | OOB interface (`vmbrOOB`) | Break-glass access only — no production traffic | 🔄 |
| 5.3 | Production interface | VLAN-tagged per service role (see topology — §5.5) | ⚠️ topology TBD |
| 5.4 | Additional interfaces | Possible for appliances (firewall, router VMs) needing multiple VLANs | ✅ |
| 5.5 | IP ranges & topology | **To be defined — see open design** below | ⚠️ |
| 5.6 | Gateway | VLAN-specific GW from topology | ⚠️ |
| 5.7 | MTU | **Must match switch config on Arista** — 1500 default, 9000 for storage VLAN | 🔄 Arista dependency |
| 5.8 | Proxmox firewall | Disabled at VM level — OPNsense handles it | 🔄 OPNsense |
| 5.9 | IPv6 | **Dual-stack by default** — per your topology decision | 🔄 confirmed |
| 5.10 | Network model | `virtio` | ✅ |
| 5.11 | PCIe / IO passthrough | Supported — GPU, PTP NIC, network cards assigned directly to VM via IOMMU | 🔄 see §7.3 |

> **5.5 Network topology — open design items (to finalize before cluster):**
> - VLAN IDs per role (MGMT, OOB, DATA, STORAGE, BACKUP)
> - IP ranges per VLAN
> - VRF assignment on Arista per VLAN
> - DNS per VLAN (see §6.8)
> - DHCP scope per VLAN (static for infra VMs, DHCP for ephemeral)
> - IPv6 prefix delegation strategy

---

## 6. Cloud-init (Linux only)

| ID | Property | Value | Status |
|---|---|---|---|
| 6.1 | User | `by-systems` | ✅ |
| 6.2 | Password | `ci_password` Terraform variable | ✅ |
| 6.3 | SSH authorized keys | **One key: `by-systems` personal key only** (see note) | 🔄 cleaned up |
| 6.4 | Upgrade packages on boot | `false` — Ansible handles | ✅ |
| 6.5 | Install packages on boot | `false` — Ansible handles | ✅ |
| 6.6 | Locale via cloud-init | `fr_BE.UTF-8` — **needs re-verification** | ⚠️ |
| 6.7 | Timezone via cloud-init | `Europe/Brussels` | ✅ |
| 6.8 | DNS nameservers | **VRF IP on Arista switch for MGMT VLAN** (not OPNsense, not 1.1.1.1) | 🔄 updated |
| 6.9 | Search domain | `by-systems.arpa` | ✅ confirmed |
| 6.10 | Cloud-init drive | `ide2` (Proxmox standard) | ✅ |

> **6.3 SSH key strategy — cleaned up:**
>
> | Key | Owner | Status |
> |---|---|---|
> | `yboujraf@personal-2026-03-27` | My Lord (Win11) | ✅ inject in all VMs |
> | `rune@by-systems-rune-vm` | Rune automation | ⚠️ rename decision needed — see below |
>
> **Open question on Rune key:** The automation key needs a cleaner identity.  
> Options: `svc-ansible@by-systems-rune` or `svc-automation@rune-vm` — to decide.  
> Until decided: inject both, clean up naming before Phase 2.
>
> **6.8 DNS architecture:**
> - **MGMT VLAN** → DNS = VRF IP on Arista switch (reaches all fabric devices)
> - **Service VLANs** → DNS = OPNsense VLAN interface IP (resolves `by-systems.arpa` + forwards external via DoT/DoH)
> - **No direct 1.1.1.1** — all external DNS must go through OPNsense DoT/DoH enforcement

---

## 7. Security Baseline (Terraform layer)

| ID | Property | Value | Status |
|---|---|---|---|
| 7.1 | QEMU guest agent | Enabled | ✅ |
| 7.2 | Serial port | **None by default** — only add for headless VMs requiring serial console (not needed for standard Linux or VGA desktop VMs) | 🔄 clarified |
| 7.3 | USB / PCIe passthrough | **Off by default.** For VMs requiring direct hardware (GPU, PTP NIC, IO card): defined explicitly per-VM via IOMMU passthrough — not in base template | 🔄 clarified |
| 7.4 | VGA type | `std` for headless infra VMs / `virtio` or `vmware` for desktop/GPU VMs | 🔄 per-VM |
| 7.5 | Protection flag | Disabled (PoC) — enable in prod | ✅ |
| 7.6 | Start on boot | Enabled | ✅ |
| 7.7 | Boot order | `scsi0` only | ✅ |

> **7.2 Serial port:** Removed from template because it caused noVNC console issues. Add back explicitly only for VMs where serial console is the primary console (embedded/network appliances). Standard Linux VMs use VGA via noVNC.
>
> **7.3 Passthrough scope:** Includes GPU, PTP NIC for timing-critical VMs, dedicated network cards for firewalls/routers, any PCIe device. Each requires IOMMU group isolation — configured per-VM, not in base template.

---

## 8. Security Hardening (Ansible layer)

Sync point: align with `odoo-install` SSH hardening role — use it as the reference baseline and extend.

| ID | Property | Value | Ansible role |
|---|---|---|---|
| 8.1 | SSH port | `22222` | `role-ssh-hardening` |
| 8.2 | SSH key-only auth | Yes — `PasswordAuthentication no` | `role-ssh-hardening` |
| 8.3 | Root SSH login | Disabled | `role-ssh-hardening` |
| 8.4 | Allowed SSH ciphers | From odoo-install baseline | `role-ssh-hardening` |
| 8.5 | ufw firewall | Enabled, default deny in/allow out | `role-ufw` |
| 8.6 | fail2ban | Enabled (SSH + service-specific) | `role-fail2ban` |
| 8.7 | NTP | **Chrony** (not systemd-timesyncd) → sync to MGMT VLAN NTP source | 🔄 `role-chrony` |
| 8.8 | Audit logging | `auditd` | `role-auditd` |
| 8.9 | Sudo | `by-systems` passwordless for Ansible | `role-base` |
| 8.10 | MOTD | **Disabled** — removes "Welcome to Ubuntu/Debian" banners. Keeps console clean, avoids leaking OS/version info to logged-in users | 🔄 clarified |
| 8.11 | Unattended-upgrades | Security patches only | `role-unattended-upgrades` |
| 8.12 | Kernel hardening | sysctl baseline (net, fs hardening) | `role-sysctl` |

> **8.7 Chrony vs systemd-timesyncd:**  
> `chrony` is preferred because it handles network interruptions better, supports hardware timestamping (future PTP), is configurable per-interface, and is the standard on RHEL/Rocky. `systemd-timesyncd` is simpler but limited — not suitable once we have fabric-level NTP (Arista as NTP server for MGMT VLAN).
>
> **8.10 MOTD:** Disabling removes:
> - OS/version banner (info leak)
> - Ubuntu/Debian "news" (useless noise)
> - Legal banners replace it if needed via `/etc/issue.net`

---

## 9. Ansible Compatibility

| ID | Property | Value | Status |
|---|---|---|---|
| 9.1 | Python3 | Included in all 3 cloud images (Debian 12, Ubuntu 24.04, Rocky 9) | ✅ |
| 9.2 | SSH port in inventory | `ansible_port: 22222` — set **after** hardening role runs | ✅ |
| 9.3 | Bootstrap sequence | Port 22 → run hardening → port changes to 22222 → all subsequent plays use 22222 | 🔄 clarified |
| 9.4 | Privilege escalation | `become: yes` + `sudo` passwordless for `by-systems` | ✅ |
| 9.5 | Inventory Phase 1 | Static `hosts.yml` per environment | ✅ |
| 9.6 | Inventory Phase 2+ | NetBox dynamic inventory plugin (`netbox.netbox.nb_inventory`) | ⚠️ blocked on NetBox |
| 9.7 | Connection type Linux | `ssh` | ✅ |
| 9.8 | Connection type Windows | `winrm` — separate inventory group, deferred Phase 4 | ⚠️ deferred |
| 9.9 | OS-specific roles | Separate role paths per OS family: `debian/`, `rhel/`, `ubuntu/` | ⚠️ structure to define |
| 9.10 | Synology lib integration | `lib-synology-dsm` called via Ansible `uri` module or Python task — not a native Ansible module yet | ⚠️ design needed |

> **9.3 Bootstrap sequence (important):**
> ```
> Day 0: Terraform provisions VM (SSH on port 22, key auth)
> Day 1: Ansible bootstrap play → runs on port 22
>   - Applies role-base (user, sudo)
>   - Applies role-ssh-hardening (changes port to 22222, key-only)
> Day 2+: All Ansible plays use port 22222
> ```
> Inventory has two groups: `bootstrap` (port 22) and `hardened` (port 22222). A VM moves from one to the other after first hardening run.

---

## 10. Role-based Sizing Defaults

Services requiring PostgreSQL + Redis run those as dedicated shared VMs, not embedded per service.

| ID | Role | CPU cores | RAM (MB) | Disk (GB) | Storage | Notes |
|---|---|---|---|---|---|---|
| 10.1 | Utility / bootstrap | 1 | 1024 | 10 | poc-data | Temporary VMs |
| 10.2 | NetBox | 2 | 4096 | 30 | poc-data | Needs PostgreSQL + Redis (shared) |
| 10.3 | Vault | 2 | 2048 | 20 | poc-data | |
| 10.4 | Authentik | 2 | 4096 | 20 | poc-data | Needs PostgreSQL + Redis (shared) |
| 10.5 | GitLab CE | 4 | 8192 | 100 | poc-data | Bundled Postgres (PoC), external in prod |
| 10.6 | GitLab Runner | 2 | 4096 | 50 | poc-data | |
| 10.7 | Nexus OSS | 2 | 4096 | 100 | poc-data | |
| 10.8 | **OPNsense** (not pfSense) | 2 | 2048 | 10 | poc-data | Full API + Ansible support | 
| 10.9 | PostgreSQL (shared) | 2 | 4096 | 50 | poc-data | Shared by NetBox, Authentik, future |
| 10.10 | Redis (shared) | 1 | 2048 | 10 | poc-data | Shared cache/queue |
| 10.11 | K3s / K8s node | 4 | 8192 | 80 | poc-data | On 2nd Proxmox node |
| 10.12 | Monitoring (Grafana/Loki) | 2 | 4096 | 50 | poc-data | |

> **10.8 OPNsense vs pfSense:**  
> OPNsense is the correct choice. Reasons:
> - Full REST API (`/api/v1/`) — every config is programmable
> - Official Ansible collection (`ansibleguy.opnsense`)
> - Active development, more frequent security updates
> - Better plugin ecosystem (Zenarmor, Tailscale, WireGuard built-in)
> - pfSense API is incomplete, community-maintained only
>
> **10.9/10.10 PostgreSQL + Redis as shared services:**  
> Rather than embedding a DB in each service VM, run one PostgreSQL VM and one Redis VM shared across NetBox, Authentik, and future services. This is cleaner operationally. In cluster mode (3 VMs), these become HA pairs.

---

## 11. Tagging & Metadata Flow

| ID | Property | Value | Status |
|---|---|---|---|
| 11.1 | Proxmox tags | `env:poc`, `role:<service>`, `managed:terraform`, `os:<debian12|ubuntu2404|rocky9>` | ⚠️ not yet applied |
| 11.2 | VM description | `<service> — IaC managed. Ref: github.com/by-openclaw/platform-setup#<issue>` | ⚠️ not yet applied |
| 11.3 | NetBox registration | Terraform → creates VM in NetBox via API after provision | ⚠️ blocked on NetBox |
| 11.4 | VM ID range | `100–199` PoC, `200–299` reserved | ✅ |
| 11.5 | NetBox → Ansible | NetBox dynamic inventory drives Ansible targeting (Phase 2+) | ⚠️ |

> **11 Flow — end to end:**
> ```
> Terraform plan → Proxmox VM created → Proxmox tags applied
>   → (future) NetBox VM record created via API
>   → Ansible static inventory updated manually (Phase 1)
>   → Ansible bootstrap play runs
>   → VM registered in NetBox with IP, role, status (Phase 2+)
>   → NetBox dynamic inventory picks it up for all future plays
> ```
> Until NetBox is live: static `hosts.yml` + Proxmox tags are the source of truth.

---

## 12. Open Decisions (D-series)

| ID | Question | Decision |
|---|---|---|
| D.1 | CPU type for cluster | `host` now → switch to `x86-64-v2-AES` before forming cluster unless nodes confirmed identical |
| D.2 | Storage: NAS vs Ceph | NAS (poc-data) now. Ceph at 3rd node + Arista storage VLAN configured |
| D.3 | DNS architecture | Per §6.8: MGMT → Arista VRF, Service VLANs → OPNsense. No 1.1.1.1 direct |
| D.4 | SSH port change | **Ansible only** — Terraform provisions on port 22, Ansible hardening moves to 22222. Terraform never touches day-2 security config |
| D.5 | Windows templates | Deferred Phase 4+ |
| D.6 | Network topology | Needs dedicated session — VLAN IDs, IP ranges, VRF config, dual-stack prefix |
| D.7 | Rune SSH key naming | Decision: `svc-ansible@by-systems-rune` or similar — to confirm |
| D.8 | PostgreSQL HA | Single VM (PoC) → 3-VM cluster (Phase 3) |
| D.9 | OPNsense confirmed | ✅ replacing pfSense |

---

## 13. Storage Backend Decision: NAS vs Ceph

| | Synology NAS (poc-data NFS) | Ceph |
|---|---|---|
| Nodes required | 1 Proxmox minimum | **3 nodes minimum** for quorum |
| 2-node cluster | ✅ | ⚠️ no quorum — split-brain risk |
| Setup complexity | Low | High — dedicated network, OSD disks, monitor quorum |
| Network dependency | 1GbE/10GbE to NAS | Dedicated storage network, MTU 9000 on Arista |
| HA live migration | ✅ | ✅ |
| Phase 1–2 (≤2 nodes) | **✅ Use this** | ❌ |
| Phase 3+ (3+ nodes) | Still valid | ✅ preferred |

**Sequence to reach Ceph:**
1. Arista switches → storage VLAN configured (MTU 9000)
2. 3rd Proxmox node added
3. OSD disks dedicated per node
4. Ceph cluster formed in Proxmox
5. Migrate VMs from poc-data → Ceph incrementally
