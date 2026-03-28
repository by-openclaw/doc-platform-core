# VM Template Specification

**Status:** Draft — review required  
**Last updated:** 2026-03-28  
**Scope:** All VMs and LXCs provisioned via Terraform on Proxmox PoC  
**Reviewer:** @yboujraf  

This document defines every setting a Terraform VM template must configure.  
Each property has a numeric ID so it can be referenced, confirmed, or questioned individually.

---

## 1. OS Support

| ID | Property | Linux | Windows | Decision |
|---|---|---|---|---|
| 1.1 | Supported | ✅ Primary target | ⚠️ Possible (see notes) | Linux is the default for all platform services |
| 1.2 | Base image | Debian 12 cloud image | Windows Server 2022 evaluation ISO | Debian 12 for all infra VMs |
| 1.3 | Cloud-init support | ✅ Native | ❌ Not supported | Windows requires Sysprep + WinRM bootstrap instead |
| 1.4 | Ansible management | ✅ SSH | ✅ WinRM (different inventory) | Two separate template definitions required |
| 1.5 | Template scope | `linux-debian12` | `windows-2022` (future) | Windows template deferred — not in Phase 1–3 |

> **Windows note:** Windows VMs on Proxmox require VirtIO drivers, Sysprep for cloud-init equivalent, and WinRM for Ansible. This is a separate template family. Not planned until Phase 4+ (developer workstations scope).

---

## 2. Identity & Locale

| ID | Property | Value | Confirmed? |
|---|---|---|---|
| 2.1 | Hostname pattern | `vm-<service>-<env>-<nn>` (e.g. `vm-netbox-poc-01`) | ✅ |
| 2.2 | Locale | `fr_BE.UTF-8` | ✅ validated |
| 2.3 | Timezone | `Europe/Brussels` | ✅ validated |
| 2.4 | Keyboard layout | `be` | ✅ validated |
| 2.5 | Default shell | `/bin/bash` | ✅ |
| 2.6 | Default user | `by-systems` | ✅ |
| 2.7 | Root login | Disabled via SSH | to confirm in Ansible hardening |
| 2.8 | Search domain | `by-systems.arpa` | ⚠️ not yet set in cloud-init |
| 2.9 | DNS resolver | pfSense IP (when live) / `1.1.1.1` interim | ⚠️ interim only |

---

## 3. Compute

| ID | Property | Value | Confirmed? |
|---|---|---|---|
| 3.1 | CPU type | `x86-64-v2-AES` | ⚠️ currently `host` — needs decision |
| 3.2 | CPU sockets | `1` | ✅ |
| 3.3 | CPU cores | Variable per role (see §10) | ✅ |
| 3.4 | Memory (RAM) | Variable per role (see §10) | ✅ |
| 3.5 | Memory ballooning | Disabled | ✅ recommended — predictable RAM allocation |
| 3.6 | NUMA | Disabled | ✅ for PoC single-socket hosts |
| 3.7 | CPU hotplug | Disabled | ✅ |
| 3.8 | Memory hotplug | Disabled | ✅ |

> **3.1 CPU type decision:** `host` = max performance but not live-migratable between different CPUs. `x86-64-v2-AES` = slightly lower performance but fully portable across nodes. **Recommendation: `x86-64-v2-AES`** — mandatory once cluster has 2 nodes with potentially different CPU models.

---

## 4. Storage

| ID | Property | Value | Confirmed? |
|---|---|---|---|
| 4.1 | SCSI controller | `virtio-scsi-single` | ✅ |
| 4.2 | iothread | Enabled | ✅ |
| 4.3 | Disk format (local-lvm) | `raw` | ✅ best performance on LVM |
| 4.4 | Disk format (NFS/NAS) | `qcow2` | ✅ required for snapshots on NFS |
| 4.5 | Boot disk size | Variable per role (see §10) | ✅ |
| 4.6 | Disk cache (local-lvm) | `none` | ✅ LVM handles caching |
| 4.7 | Disk cache (NFS) | `writeback` | ⚠️ needs decision |
| 4.8 | Discard / TRIM | Enabled | ✅ for thin provisioning |
| 4.9 | SSD emulation | Enabled | ✅ for virtual disk trim support |
| 4.10 | Storage backend default | `local-lvm` (PoC) | ✅ NFS for shared VMs |
| 4.11 | Backup | Enabled for all | ⚠️ backup schedule not defined yet |
| 4.12 | Snapshot policy | Manual only (PoC) | ⚠️ automated snapshots deferred |

---

## 5. Network

| ID | Property | Value | Confirmed? |
|---|---|---|---|
| 5.1 | Default bridge | `vmbrOOB` | ✅ PoC — all VMs on OOB |
| 5.2 | VLAN tag | None (untagged, PoC) | ✅ VLAN per role deferred to Phase 2 |
| 5.3 | MAC address | Auto-generated | ✅ |
| 5.4 | IP assignment | Static via cloud-init | ✅ |
| 5.5 | IP range (PoC VMs) | `10.6.225.x` | ✅ |
| 5.6 | Gateway | `10.6.255.254` (pfSense OOB) | ✅ |
| 5.7 | MTU | `1500` default | ✅ jumbo frames deferred (storage VLAN) |
| 5.8 | Proxmox firewall | Disabled at VM level | ✅ pfSense handles it |
| 5.9 | IPv6 | Deferred | ⚠️ dual-stack in Phase 2+ |
| 5.10 | Network model | `virtio` | ✅ |

---

## 6. Cloud-init (Linux only)

| ID | Property | Value | Confirmed? |
|---|---|---|---|
| 6.1 | User | `by-systems` | ✅ |
| 6.2 | Password | `ci_password` Terraform variable | ✅ |
| 6.3 | SSH authorized keys | Both keys injected (personal + rune automation) | ✅ |
| 6.4 | Upgrade packages on boot | `false` | ✅ Ansible handles updates |
| 6.5 | Install packages on boot | `false` | ✅ Ansible handles packages |
| 6.6 | Locale via cloud-init | `fr_BE.UTF-8` | ⚠️ set in template, verify in cloud-init module |
| 6.7 | Timezone via cloud-init | `Europe/Brussels` | ✅ |
| 6.8 | DNS nameservers | `10.6.255.254` (pfSense) | ⚠️ set to `1.1.1.1` interim |
| 6.9 | Search domain | `by-systems.arpa` | ⚠️ not set yet |
| 6.10 | Cloud-init drive | `ide2` | ✅ (standard Proxmox) |

---

## 7. Security Baseline (Terraform layer)

These are set by Terraform at provision time, before Ansible runs.

| ID | Property | Value | Confirmed? |
|---|---|---|---|
| 7.1 | QEMU guest agent | Enabled | ✅ |
| 7.2 | Serial port | None (removed from template) | ✅ |
| 7.3 | USB passthrough | None | ✅ |
| 7.4 | VGA type | `std` (noVNC console only) | ✅ |
| 7.5 | Protection flag | Disabled (PoC) | ✅ enable in prod |
| 7.6 | Start on boot | Enabled | ✅ |
| 7.7 | Boot order | `scsi0` only | ✅ |

---

## 8. Security Hardening (Ansible layer)

These are applied by Ansible after Terraform provisions. Listed here for completeness.

| ID | Property | Value | Ansible role |
|---|---|---|---|
| 8.1 | SSH port | `22222` | `role-ssh-hardening` |
| 8.2 | SSH key-only auth | Yes (PasswordAuthentication no) | `role-ssh-hardening` |
| 8.3 | Root SSH login | Disabled | `role-ssh-hardening` |
| 8.4 | ufw firewall | Enabled, default deny | `role-ufw` |
| 8.5 | fail2ban | Enabled (SSH) | `role-fail2ban` |
| 8.6 | unattended-upgrades | Enabled (security only) | `role-unattended-upgrades` |
| 8.7 | NTP | `systemd-timesyncd` → pfSense NTP | `role-ntp` |
| 8.8 | Audit logging | `auditd` | `role-auditd` |
| 8.9 | Sudo | `by-systems` passwordless | `role-base` |
| 8.10 | Motd | Disabled | `role-base` |

---

## 9. Ansible Compatibility

| ID | Property | Value | Confirmed? |
|---|---|---|---|
| 9.1 | Python3 on image | Included in Debian 12 cloud image | ✅ |
| 9.2 | SSH port in inventory | `ansible_port: 22222` | ⚠️ requires Ansible inventory update after hardening |
| 9.3 | Privilege escalation | `become: yes` / `sudo` passwordless | ✅ |
| 9.4 | Inventory source (Phase 1) | Static `hosts.yml` | ✅ temporary |
| 9.5 | Inventory source (Phase 2+) | NetBox dynamic inventory plugin | ⚠️ blocked on NetBox deploy |
| 9.6 | Connection type | `ssh` | ✅ |
| 9.7 | WinRM (Windows) | `winrm` — separate inventory group | ⚠️ deferred Phase 4+ |

---

## 10. Role-based Sizing Defaults

| ID | Role | CPU cores | RAM (MB) | Disk (GB) | Storage |
|---|---|---|---|---|---|
| 10.1 | Utility / bootstrap | 1 | 1024 | 10 | local-lvm |
| 10.2 | NetBox | 2 | 4096 | 50 | local-lvm |
| 10.3 | Vault | 2 | 2048 | 20 | local-lvm |
| 10.4 | Authentik | 2 | 4096 | 30 | local-lvm |
| 10.5 | GitLab CE | 4 | 8192 | 100 | NAS NFS |
| 10.6 | GitLab Runner | 2 | 4096 | 50 | local-lvm |
| 10.7 | Nexus OSS | 2 | 4096 | 100 | NAS NFS |
| 10.8 | pfSense | 2 | 2048 | 10 | local-lvm |
| 10.9 | K3s node | 4 | 8192 | 80 | local-lvm |
| 10.10 | Monitoring (Grafana/Loki) | 2 | 4096 | 50 | NAS NFS |

---

## 11. Tagging & Metadata

| ID | Property | Value | Confirmed? |
|---|---|---|---|
| 11.1 | Proxmox tags | `env:poc`, `role:<service>`, `managed:terraform` | ⚠️ not yet applied |
| 11.2 | Description field | Service name + GitHub issue reference | ⚠️ not yet applied |
| 11.3 | NetBox sync | VM registered in NetBox after create | ⚠️ blocked on NetBox |
| 11.4 | VM ID range | `100–199` PoC, `200–299` reserved | ✅ |

---

## Open Decisions

| ID | Question | Options | Recommendation |
|---|---|---|---|
| D.1 | CPU type: `host` vs `x86-64-v2-AES` | host=fast, x86-64-v2=portable | **x86-64-v2-AES** once cluster formed |
| D.2 | Storage: NAS NFS vs Ceph | See §12 | NAS now, Ceph on 3rd node |
| D.3 | DNS: interim `1.1.1.1` vs pfSense | pfSense not live yet | pfSense first, then update cloud-init |
| D.4 | SSH port: change at Terraform or Ansible | Ansible is cleaner | Ansible (day-2 hardening role) |
| D.5 | Windows template scope | Defer to Phase 4 | ✅ confirmed deferred |

---

## 12. Storage Backend Decision: NAS vs Ceph

| | Synology NAS (NFS/iSCSI) | Ceph (Proxmox built-in) |
|---|---|---|
| **Nodes required** | 1 Proxmox node minimum | **Minimum 3 nodes** for quorum |
| **2-node cluster** | ✅ Works fine | ⚠️ Possible but no quorum — split-brain risk |
| **Setup complexity** | Low — NFS share + Proxmox storage config | High — dedicated network, OSD disks, monitor quorum |
| **Performance** | Network-bound (1GbE/10GbE) | Near-local disk speed (dedicated storage network) |
| **HA live migration** | ✅ Shared storage → live migration works | ✅ Same |
| **Cost** | NAS already exists ✅ | Requires dedicated disks per node |
| **Phase 1–2 (2 nodes)** | **✅ Recommended** | ❌ Not recommended |
| **Phase 3+ (3+ nodes)** | Still valid | ✅ Preferred long-term |

**Decision for tomorrow's cluster:**
- Use **Synology NAS** as shared storage for the 2-node Proxmox cluster
- Ceph becomes relevant when a **3rd node** is added
- Ceph also needs a **dedicated storage VLAN** (jumbo frames, MTU 9000) — that requires the Arista switches to be configured first

**What Ceph needs before it can be considered:**
1. 3rd Proxmox node
2. Dedicated storage NIC per node (or VLAN on Arista)
3. At least 1 dedicated OSD disk per node (not the OS disk)
4. Storage VLAN configured on Arista (MTU 9000)

So the sequence is: **Arista switches → storage VLAN → 3rd node → Ceph**. Not tomorrow.
