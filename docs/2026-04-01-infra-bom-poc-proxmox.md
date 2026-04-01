# Bill of Materials - PoC Proxmox Infra Deploy

> **Date:** 2026-04-01
> **Author:** Rune
> **Status:** DRAFT - @yboujraf reviews + approves → Opus validates → ADR
> **Hardware:** srv-proxmox-poc-01 - 12 cores, **188 GB RAM**, NAS storage (Synology DS1513+)
> **Network:** `10.1.0.0/20` PoC supernet - **isolated from prod `10.6.0.0/20`**. OOB dependency: Proxmox host management + NAS NFS host-level mounts (`poc-iso`, `poc-backup`) only. VMs do NOT mount NFS directly. No PoC VM traffic routes to OOB NAS at runtime.
> **WAN:** OPNsense connects directly to both ISPs via 3com OOB switch - Proximus (PPPoE VLAN 10) + Telenet (static VLAN 999). Primary OOB prod confirmed working. Telenet as PoC WAN is **untested** - two firewalls (prod pfSense + PoC OPNsense) sharing same Telenet gateway is unknown territory. Must verify before relying on it. WireGuard endpoint on Telenet fixed IP is the target, not confirmed.
> **VLAN IDs 300-series (PoC infra), 400-series (PoC users). Verified against live switch configs (2026-04-01).**
> **Rules:** Docker-only, Traefik as reverse proxy, no Apache/nginx. Cluster-ready (Docker Swarm → K3s path). Minimum VM sizes set conservatively - bump as needed (167 GB free RAM, no constraints).
> **Base image:** Debian 12 cloud-init template

---

## Source Repos & Docker Images

> All tools are self-hosted via Docker. No SaaS. Images pulled from Docker Hub / GitHub Container Registry at deploy time.

| VM | Tool | Source / Repo | Docker Image |
|---|---|---|---|
| vm-opnsense-poc-01 | OPNsense 24.x | <https://github.com/opnsense/core> · <https://opnsense.org/download/> | ISO - bare VM, not Docker |
| vm-pihole-poc-01 | Pi-hole v6 | <https://github.com/pi-hole/pi-hole> · <https://pi-hole.net> | `pihole/pihole:latest` |
| vm-traefik-poc-01 | Traefik v3 | <https://github.com/traefik/traefik> · <https://traefik.io> | `traefik:v3` |
| vm-vault-poc-01 | HashiCorp Vault | <https://github.com/hashicorp/vault> · <https://developer.hashicorp.com/vault> | `hashicorp/vault:latest` |
| vm-vaultwarden-poc-01 | Vaultwarden | <https://github.com/dani-garcia/vaultwarden> | `vaultwarden/server:latest` |
| vm-authentik-poc-01 | Authentik | <https://github.com/goauthentik/authentik> · <https://goauthentik.io> | `ghcr.io/goauthentik/server:latest` |
| vm-gitlab-poc-01 | GitLab CE | <https://gitlab.com/gitlab-org/gitlab> · <https://about.gitlab.com/install/> | `gitlab/gitlab-ce:latest` |
| vm-gitlab-runner-poc-01 | GitLab Runner | <https://gitlab.com/gitlab-org/gitlab-runner> | `gitlab/gitlab-runner:latest` |
| vm-nextcloud-poc-01 | Nextcloud | <https://github.com/nextcloud/server> · <https://nextcloud.com> | `nextcloud:fpm-alpine` |
| vm-netbox-poc-01 | NetBox | <https://github.com/netbox-community/netbox> · <https://netbox.dev> | `ghcr.io/netbox-community/netbox:latest` |
| vm-nexus-poc-01 | Nexus OSS | <https://github.com/sonatype/nexus-public> · <https://www.sonatype.com/products/sonatype-nexus-oss> | `sonatype/nexus3:latest` |
| vm-observability-poc-01 | Prometheus | <https://github.com/prometheus/prometheus> | `prom/prometheus:latest` |
| vm-observability-poc-01 | Grafana | <https://github.com/grafana/grafana> | `grafana/grafana:latest` |
| vm-observability-poc-01 | Loki | <https://github.com/grafana/loki> | `grafana/loki:latest` |
| vm-postgres-poc-01 | PostgreSQL 16 | <https://github.com/postgres/postgres> | `postgres:16-alpine` |
| vm-redis-poc-01 | Redis ≤7.2 (BSD-3) | <https://github.com/redis/redis> | `redis:7.2.7-alpine` ⚠️ pin — do not upgrade to 7.4+ (RSALv2/SSPL) |
| vm-unifi-poc-01 | Unifi Network App | <https://github.com/linuxserver/docker-unifi-network-application> | `lscr.io/linuxserver/unifi-network-application:latest` |

---

## Free / Community Edition Limitations - PoC Fitness Check

> Verified 2026-04-01. All tools used are free/open-source editions.

| Tool | Edition | License | Key Limitations vs Paid | PoC Fit? |
|---|---|---|---|---|
| **OPNsense** | Community | BSD 2-clause | No commercial support. Business Edition adds ZenArmor NGFW, netdata plugin, config sync. All routing/FW/WireGuard/VPN features included free. | ✅ Full fit |
| **Pi-hole** | Free / OSS | EUPL 1.2 | No paid tier exists. Fully free forever. Phase 2 → BIND9. | ✅ Full fit |
| **Traefik** | OSS (v3) | MIT | Traefik Enterprise adds distributed rate-limiting, multi-cluster, RBAC, WAF. OSS has everything needed: TLS, Let's Encrypt, middleware, routing. | ✅ Full fit |
| **HashiCorp Vault** | Community | BSL 1.1 | **BSL since Aug 2023.** Free for internal use + non-competing products. Missing vs Enterprise: Namespaces, HSM auto-unseal, DR replication, Secrets Sync, SAML auth, FIPS 140-2, Sentinel, Transform secrets engine. Core secrets management fully functional. | ✅ Full fit for PoC. Note BSL for prod compliance review. |
| **Vaultwarden** | Free / OSS | AGPL 3.0 | Unofficial Bitwarden API reimplementation. Not Bitwarden official. No paid tier. All Bitwarden client features work (passwords, 2FA, collections). | ✅ Full fit |
| **Authentik** | OSS (self-hosted) | MIT | Enterprise adds: support SLA, FIPS compliance, volume pricing. **All protocols free: OIDC, SAML, LDAP, SCIM, RADIUS, Kerberos, RAC (RDP/SSH - moved to OSS in 2025.2).** No feature gating on OSS self-hosted. Confirmed: no features will ever be moved from OSS to Enterprise. | ✅ Full fit |
| **GitLab CE** | Community Edition | MIT | Missing vs EE: SAML SSO, LDAP group sync, Merge Trains, Code Owners (enforcement), Audit Events (advanced), Container Scanning, DAST, Compliance Frameworks, Epics, Portfolio Mgmt. **No user/project/pipeline limits on CE.** Repos, CI/CD, Registry all unlimited. | ✅ Full fit for PoC. Note: SAML SSO handled by Authentik (OIDC) - not a gap. |
| **Nexus OSS** | Community Edition (CE) | EPL 1.0 | Missing vs PRO: SAML, HA deployment, Azure/GCS blob stores, staging & build promotion, content replication, User Tokens, Docker subdomain connector. **AWS S3 blob store: ✅ included in CE (Contabo S3 compatible).** All format support included (npm, pip, Docker, Helm, Maven, PyPI, etc.). | ✅ Full fit. S3 backend confirmed free. |
| **Nextcloud** | Community | AGPL 3.0 | Enterprise adds: official support, Nextcloud Office (Collabora), Groupware push notifications, legal hold, e-discovery, certified apps. Core file sync + S3 storage + OIDC SSO fully free. | ✅ Full fit |
| **NetBox** | OSS | Apache 2.0 | Enterprise/Cloud adds: support SLA, high-availability, SSO (advanced), some add-ons under Polyform Shield. **Core IPAM/DCIM fully free, unlimited devices/IPs/prefixes.** OIDC SSO via Authentik works on OSS. | ✅ Full fit |
| **Prometheus** | OSS | Apache 2.0 | No paid tier. Fully free. HA/remote write → Thanos/Cortex for scale (not needed for PoC). | ✅ Full fit |
| **Grafana** | OSS | AGPL 3.0 | Enterprise adds: enhanced LDAP, reporting, data source permissions, auditing. OSS covers all dashboards, alerting, Loki/Prometheus datasources. | ✅ Full fit |
| **Loki** | OSS | AGPL 3.0 | Enterprise adds: support, multi-tenancy, FIPS. OSS with S3 backend covers all PoC log ingestion. | ✅ Full fit |
| **PostgreSQL** | OSS | PostgreSQL License | No paid tier. Fully free. | ✅ Full fit |
| **Redis** | OSS (v7) | RSALv2 + SSPL | **License changed to RSALv2 + SSPL in Redis 7.4+** - not OSI-approved. Use `redis:7.2-alpine` (BSD licensed, last BSD release). For PoC = no practical impact. | ✅ Full fit. Pin to `redis:7.2-alpine`. |
| **Unifi Network App** | Free | Proprietary (Ubiquiti) | No subscription needed for self-hosted controller. All WiFi/VLAN/AP management features included. Cloud key not required. | ✅ Full fit |

### Verdict

**All tools fit PoC on free/community editions. Zero blockers.**

Two items to note for Phase 2 / prod compliance:
- **Vault BSL:** review BSL 1.1 terms before production deployment in customer-facing context. Internal use is unambiguously permitted.
- **Redis 7.4+ license:** pin Docker image to `redis:7.2-alpine` (BSD-licensed). No functional impact.

---

## Network Architecture (Proxmox SDN)

No physical switch between VMs - all networking is **Proxmox SDN** (VNets + Zones). OPNsense is the virtual router/firewall between zones.

```
[ OOB network 10.6.224.x ] ─── WAN ─── [ OPNsense VM ]
                                              |
                    ┌─────────────────────┼─────────────────────┐
                LAN/MGMT zone               DMZ zone              SVC VLANs
                (infra VMs)            (Traefik, WireGuard,    (per-service isolation)
             10.1.1.0/24 (MGMT vlan110)              Pi-hole, Unifi)          10.1.2.0/24 (DMZ vlan120)

[ Traefik ] ← only DMZ VM exposed to WAN → proxies HTTP/S to LAN/SVC VMs
[ WireGuard ] ← UDP 51820 exposed to WAN → VPN into LAN/MGMT
[ Pi-hole ] ← DNS :53 only from LAN/SDN zones
```

**SDN zones (Proxmox VNets) - `10.1.0.0/20` supernet, /24 per zone:**

| VLAN | VNet name | Subnet | Purpose | External access |
|---|---|---|---|---|
| **300** | `vnet-poc-oob` | 10.1.0.0/24 | OOB - Proxmox IPMI, switch mgmt ports | Physical uplink |
| **310** | `vnet-poc-mgmt` | 10.1.1.0/24 | MGMT - Rune VM (.5), Terraform, Ansible | WireGuard only |
| **320** | `vnet-poc-dmz` | 10.1.2.0/24 | DMZ - Traefik (.10), WireGuard (.61) | Traefik :443, WG :51820 |
| **330** | `vnet-poc-svc` | 10.1.3.0/24 | SVC - all service VMs | Via Traefik only |
| **340** | `vnet-poc-storage` | 10.1.4.0/24 | **RESERVED / NOT USED** - NAS NFS is host-level only (`poc-iso`, `poc-backup`). VMs never mount NFS directly. 340 has no current use case. Remove from Arista trunk allowed list until repurposed. | None |
| **350** | `vnet-poc-backup` | 10.1.5.0/24 | **RESERVED / NOT USED** - same rationale as 340. No VM-level backup network needed. Proxmox backup jobs use host-level NFS directly. | None |
| **400** | `vnet-poc-wifi` | 10.1.6.0/24 | WIFI - Unifi AP users | None |
| **410** | `vnet-poc-users` | 10.1.7.0/24 | USERS - workstations (future) | None |

**VLAN ID policy:** 300-series = PoC infra, 400-series = PoC users.
**Verified clear of:** ISP 10/20 (Proximus), P2P fabric 110-121 (Arista 7060+7020), broadcast 600-999 (SMPTE/PTP/MGMT_CTRL), Telenet 999.

**VLAN 1 policy:** Never used for real traffic. Default/native VLAN - leave dead. Vulnerable to VLAN hopping attacks (double-tagging). On Proxmox SDN: non-issue (all traffic tagged). On Arista trunk (when wired): set `native vlan 999` (blackhole) + explicitly exclude VLAN 1 from allowed list.

**Arista trunk config (when physically wired):**
```
switchport trunk native vlan 999
switchport trunk allowed vlan 300,310,320,330,340,350,400,410
! VLAN 1 intentionally absent
```

**Arista readiness:** VLAN IDs 300/310/320/330/340/350/400/410 must be added to Arista VLAN database and permitted on Proxmox trunk port when physically wired. For now (SDN-only) these are pure Proxmox VNets - IDs pre-reserved.

---

## vmbrMGMT → vmbrFAB — Fabric Control Link (Disabled for PoC)

> **Status: PENDING MANUAL ACTION — must be done before PoC deploy.**
> Three-step isolation: rename `vmbrMGMT` → `vmbrFAB` + disable bridge + disable physical NIC port.

**Purpose:** `vmbrMGMT` is the Proxmox host's connection to the Arista fabric control-plane VLAN (VLAN 600). Physical link is a **10G SFP** NIC (`nic4` / `enx0060dd44ecce`) uplinked to the Arista switch fabric. Used for switch management, SMPTE, PTP, and broadcast control traffic in prod.

**Live config (as-built on srv-proxmox-poc-01):**
```
Interface:    vmbrMGMT
Type:         Linux bridge
Physical NIC: nic4 (enx0060dd44ecce) — 10G SFP
VLAN:         nic4.600 (VLAN ID 600)
IP:           10.6.255.250/20
Comment:      MGMT / MGMT Untagged Vlan 600
Bridge STP:   off, FD: 0
Autostart:    yes (currently active on host)
```

**Why disabled for PoC:** VLAN 600 is prod fabric control plane (Arista 7060/7020 SMPTE/PTP/MGMT_CTRL range 600–999). PoC has no active fabric wiring. Risk of traffic injection into prod broadcast control plane via shared Arista switch.

**Decision (2026-04-01 @yboujraf):** Three actions required — all three before PoC terraform apply:

1. **Rename** `vmbrMGMT` → **`vmbrFAB`** — supervision/orchestration bridge (Ansible, Terraform, Prometheus → fabric devices). vmbrFAB is the PROD bridge name in the VLAN plan; disabled until PoC fabric is physically wired.
2. **Disable the bridge** — `autostart=no` in `/etc/network/interfaces`, bring down with `ifdown`
3. **Disable the physical NIC port** (`nic4` / `enx0060dd44ecce`) — hard isolation at NIC level, not just bridge software

**Manual steps on srv-proxmox-poc-01 (SSH or Proxmox console):**
```bash
# Step 1 — Bring down the bridge and VLAN interface
ifdown vmbrMGMT
ifdown nic4.600

# Step 2 — Disable physical NIC
ip link set enx0060dd44ecce down

# Step 3 — Edit /etc/network/interfaces
# - Rename 'vmbrMGMT' → 'vmbrFAB'
# - Remove 'auto vmbrFAB' line (or comment out) so it does not start on boot
# - Add: post-down ip link set enx0060dd44ecce down

# Step 4 — Verify bridge is gone from Proxmox UI (Datacenter → Node → Network)
```

**Future re-enable (when wiring PoC to fabric):**
1. Assign new bridge name per VLAN plan
2. Re-enable `nic4` (`enx0060dd44ecce`) and VLAN subinterface
3. Verify Arista port: `switchport mode trunk`, VLAN 600 allowed
4. Confirm IP assignment (10.6.255.250/20 or reassigned to PoC range)
5. Update OPNsense firewall rules for fabric VLAN access from MGMT zone

**Note:** `vmbrAPPS` (VLAN-aware bridge, no ports assigned) is also a placeholder. Kept defined, no ports, no IP. Future use: app-facing bridge if SDN VNets are insufficient.

**OPNsense WAN migration - phased (safety-first):**

| Phase | WAN1 | WAN2 | OOB role | Notes |
|---|---|---|---|---|
| **1 (now)** | Proximus PPPoE (primary) | OOB 10.6.224.x (fallback/safety) | Proxmox host mgmt + OPNsense fallback | Recovery path always available |
| **2 (stable)** | Telenet fixed (primary, WG endpoint) | Proximus PPPoE (ISP failover) | Proxmox host mgmt only | **Telenet untested** - must verify dual-FW sharing same gateway before this phase |
| **3 (target)** | Telenet fixed | Proximus PPPoE | Proxmox host mgmt only - never PoC WAN | Full ISP isolation - only after phase 2 verified |

> **Rule:** OOB `10.6.224.x` must **always** remain reachable for Proxmox host management regardless of OPNsense state. It is the recovery path. Never route it away.

> **Phase 1 gateway policy:** WAN1 Proximus weight=1 (preferred). WAN2 OOB weight=100 (fallback only - no traffic unless WAN1 down).

**Rune VM access:**
- **Phase 1:** WireGuard over internet to Proximus WAN (DHCP - use DDNS or OPNsense DynDNS). Peer IP: `10.1.1.5`.
- **Phase 2:** WireGuard endpoint migrated to Telenet fixed IP. Stable, no DDNS.
- **Phase 3 (Rune migrates to PoC):** Rune VM native on `vnet-poc-mgmt` `10.1.1.5`. WireGuard peer retired. Direct L2. This gives Rune direct access to all MGMT zone VMs without traversing OPNsense firewall rules. Rune is trusted infra, not a remote user - it belongs in MGMT zone, not behind a firewall hop.

**Rule:** VMs in `vnet-mgmt` are invisible to WAN by default. OPNsense firewall rules control what crosses zones. Traefik is the only HTTP/S gateway.

```
ISP:       3com Gi1/0/x (free port) → PoC Proxmox NIC → VLAN-aware bridge
             VLAN 10  → OPNsense vtnet0 → PPPoE Proximus (DHCP IPv4/IPv6) [WAN1]
             VLAN 999 → OPNsense vtnet1 → Telenet static IPv4/IPv6        [WAN2 primary]
HTTP/S:    Internet → OPNsense dual-WAN → Traefik (10.1.2.10 DMZ) → SVC (10.1.3.x)
VPN:       Rune VM → WireGuard@OPNsense (Telenet fixed IP) → MGMT/SVC zones
           @yboujraf → WireGuard@OPNsense (Telenet fixed IP) → MGMT/SVC zones
DNS:       VMs → OPNsense NAT :53 → Pi-hole (10.1.1.60) → OPNsense Unbound DoT :853 → 1.1.1.1
Storage:   Proxmox host → NAS (host-level NFS: poc-iso, poc-backup). VMs → Contabo S3 (runtime data).
Isolation: prod OOB (10.6.224.x) NEVER touched by PoC traffic
Migration: WireGuard endpoint = Telenet fixed IP - stable, no DDNS
```

**3com config change (one-time, non-disruptive):**
```
interface GigabitEthernet1/0/5   ! free port - verify with lldp/status
 port link-type hybrid
 undo port hybrid vlan 1
 port hybrid vlan 10 999 tagged
 stp edged-port enable
```

---

## VM Spec Sheet

> **Spec methodology:** PoC = lean but functional, no OOM risk, Docker overhead included (~256 MB RAM, ~5 GB disk base).
> Prod = realistic for a 5-10 operator team. Disk includes OS + container images + ephemeral data.

| VM | Tool(s) | PoC: vCPU / RAM / Disk | Prod: vCPU / RAM / Disk | Vendor Min RAM | Notes |
|---|---|---|---|---|---|
| vm-opnsense-poc-01 | OPNsense 24.x | 2 / 2 GB / 20 GB | 2 / 4 GB / 20 GB | 1 GB | Current sizing confirmed. Dual-WAN + WireGuard plugin on 2 GB is stable. |
| vm-pihole-poc-01 | Pi-hole v6 (Docker) | 1 / 512 MB / 10 GB | 1 / 1 GB / 10 GB | 128 MB | Current sizing confirmed. 512 MB includes OS + Docker overhead comfortably. |
| vm-traefik-poc-01 | Traefik v3 (Docker) | 1 / 1 GB / 10 GB | 2 / 2 GB / 10 GB | 64 MB | Current sizing confirmed. Generous for single-node PoC; headroom for TLS cert ops. |
| vm-vault-poc-01 | HashiCorp Vault (Docker, Raft backend) | 1 / 2 GB / 20 GB | 2 / 4 GB / 20 GB | 1 GB | **BUMPED from 1 GB → 2 GB.** Vault JVM + Raft + Docker/OS overhead OOM at 1 GB under load. |
| vm-vaultwarden-poc-01 | Vaultwarden (Docker) | 1 / 512 MB / 10 GB | 1 / 1 GB / 10 GB | 64 MB | **BUMPED from 256 MB → 512 MB.** 256 MB is zero-headroom after Docker daemon + OS (~256 MB baseline). |
| vm-authentik-poc-01 | Authentik (server+worker+Redis+PostgreSQL) | 2 / 2 GB / 20 GB | 4 / 4 GB / 20 GB | 2 GB | Current sizing confirmed. Vendor-confirmed 2 GB minimum - OOMs below this on startup. |
| vm-gitlab-poc-01 | GitLab CE (Puma+Sidekiq+nginx+PostgreSQL+Redis) | 4 / 8 GB / 50 GB | 8 / 16 GB / 100 GB | 4 GB (8 GB recommended) | Current sizing confirmed. Constraint-held at 4 vCPU / 8 GB minimum per task spec. |
| vm-gitlab-runner-poc-01 | GitLab Runner (Docker executor) | 2 / 2 GB / 20 GB | 4 / 4 GB / 30 GB | 1 GB | Current sizing confirmed. Light PoC pipelines (lint/build/test). Scale cores for parallel CI. |
| vm-nextcloud-poc-01 | Nextcloud-fpm + nginx sidecar (Docker), S3 primary storage | 2 / 2 GB / 20 GB | 4 / 4 GB / 30 GB | 512 MB | Current sizing confirmed. fpm workers + nginx sidecar + S3 client fit in 2 GB. |
| vm-netbox-poc-01 | NetBox (netbox+worker) - shared postgres+redis | 2 / 2 GB / 20 GB | 2 / 4 GB / 20 GB | 512 MB | Current sizing confirmed. Django + background worker comfortable in 2 GB. |
| vm-nexus-poc-01 | Nexus OSS (Docker) | 2 / 6 GB / 50 GB | 4 / 8 GB / 100 GB | 4 GB | **⚠️ CRITICAL: BUMPED from 2 GB → 6 GB.** JVM default heap: -Xms2703m -Xmx2703m + MaxDirectMemory 2703m = ~5.4 GB JVM alone. 2 GB = guaranteed OOM on startup. |
| vm-observability-poc-01 | Prometheus + Grafana + Loki (colocated) | 2 / 4 GB / 30 GB | 4 / 8 GB / 50 GB | 1 GB | Current sizing confirmed. 3 colocated services: Prometheus ~256 MB + Grafana ~256 MB + Loki ~512 MB + Docker/OS. 4 GB gives query headroom with 13 VMs shipping logs. |
| vm-postgres-poc-01 | PostgreSQL 16 (shared: Authentik, NetBox, Nextcloud, Vaultwarden) | 2 / 4 GB / 50 GB | 4 / 8 GB / 100 GB | 256 MB | Current sizing confirmed. 4 active databases; shared_buffers ~1 GB + connection overhead. |
| vm-redis-poc-01 | Redis 7 (shared: Authentik, NetBox, Nextcloud) | 1 / 1 GB / 10 GB | 2 / 2 GB / 10 GB | 128 MB | **REDUCED from 2 GB → 1 GB.** In-memory store, ~50-100 MB baseline at PoC load. 1 GB gives 10x headroom. |
| vm-unifi-poc-01 | Unifi Network App (Docker) | 1 / 2 GB / 10 GB | 2 / 4 GB / 20 GB | 1 GB | **BUMPED from 1 GB → 2 GB.** Java app + bundled MongoDB require ~1.5 GB minimum. 1 GB = OOM risk on startup. |

### PoC Resource Totals (finalized)

| Metric | Finalized | Physical | Notes |
|---|---|---|---|
| vCPU total | 28 vCPU | 12 physical cores | Overcommit ~2.3x - fine, workloads are I/O-bound and mostly idle |
| RAM total | ~41 GB | **188 GB** | **147 GB free - no constraints** |
| Disk total | ~340 GB | `poc-data` ZFS pool | Thin provisioned |

### VMs with sizing changes vs. original BoM

| VM | Change | Risk if unchanged |
|---|---|---|
| vm-vault-poc-01 | 1 GB → **2 GB** | OOM under load (Raft + JVM overhead) |
| vm-vaultwarden-poc-01 | 256 MB → **512 MB** | Zero headroom after Docker/OS - guaranteed OOM |
| vm-nexus-poc-01 | 2 GB → **6 GB** | **CRITICAL** - JVM heap alone needs ~5.4 GB, OOM on startup |
| vm-redis-poc-01 | 2 GB → **1 GB** | Over-provisioned - reclaimed 1 GB, no risk |
| vm-unifi-poc-01 | 1 GB → **2 GB** | OOM risk - Java + MongoDB ~1.5 GB minimum |

---

## VM Bill of Materials

Grouped by deployment priority (Layer model - ADR-0006).

### Layer 0 - Network Foundation (OPNsense + Traefik)

| VM name | Tool | vCPU | RAM | Disk | IP | Notes |
|---|---|---|---|---|---|---|
| vm-opnsense-poc-01 | OPNsense | 2 | 2 GB | 20 GB | OOB/WAN: 10.1.0.1 / MGMT: 10.1.1.1 / DMZ: 10.1.2.1 / SVC: 10.1.3.1 | **Deploy first.** Virtual router + firewall. WAN = OOB uplink. LAN = MGMT zone. DMZ zone for Traefik/WireGuard. Proxmox SDN VNets replace physical switch. |
| vm-traefik-poc-01 | Traefik v3 | 1 | 1 GB | 10 GB | 10.1.2.10 (VLAN 120 DMZ) | Docker. Sole HTTP/S entry point. Sits in DMZ zone. Routes to MGMT zone services. TLS via LE + Cloudflare DNS-01. **GitLab TLS mode: HTTP proxy (Traefik terminates TLS, proxies HTTP to GitLab:8080 internally) - SNI passthrough rejected (requires GitLab to manage its own cert).** |

**Deploy order:** OPNsense first (SDN + firewall rules) → Traefik (DMZ) → all other services (MGMT zone).

---

### Layer 1 - Identity & Secrets (Vault + Vaultwarden + Authentik)

Deploy in order: Vault first → Authentik depends on it for secrets.

| VM name | Tool | vCPU | RAM | Disk | IP | Notes |
|---|---|---|---|---|---|---|
| vm-vault-poc-01 | HashiCorp Vault | 1 | 1 GB | 20 GB | 10.1.3.11 | Docker. Raft storage backend. Unsealing: manual for PoC. Machine secrets - API tokens, infra creds, injected into containers. |
| vm-vaultwarden-poc-01 | Vaultwarden | 1 | 256 MB | 10 GB | 10.1.3.13 | Docker. Bitwarden-compatible. Human credential management - passwords, 2FA seeds, shared team vault. Lightweight single container. Added 2026-04-01. |
| vm-authentik-poc-01 | Authentik | 2 | 2 GB | 20 GB | 10.1.3.12 | Docker (server + worker + Redis + PostgreSQL). OIDC SSO broker. Federates with **EntraID via OIDC** - EntraID is upstream IdP, Authentik adds internal RBAC + groups. All platform services authenticate via Authentik. Added 2026-04-01. |

**Two vaults, two purposes:**
- **HashiCorp Vault** = machine secrets (API tokens, TLS certs, infra credentials - consumed by automation)
- **Vaultwarden** = human credentials (passwords, 2FA, shared team secrets - accessed via Bitwarden clients)

**Authentik resource note:** Authentik is resource-heavy for its size - needs 2 GB minimum or it OOMs on startup.

---

### Layer 2 - VCS & CI (GitLab + Runner)

| VM name | Tool | vCPU | RAM | Disk | IP | Notes |
|---|---|---|---|---|---|---|
| vm-gitlab-poc-01 | GitLab CE | 4 | 8 GB | 50 GB | 10.1.3.20 | **Exception: GitLab ships its own nginx bundled.** Cannot be replaced with Traefik internally. Traefik sits in front as TCP passthrough on port 443. All other tools use Docker + Traefik. |
| vm-gitlab-runner-poc-01 | GitLab Runner | 2 | 2 GB | 20 GB | 10.1.3.21 | Docker executor. Runs CI pipelines. No web exposure. |

**GitLab exception note:** GitLab CE bundles nginx and Puma - it cannot run cleanly with an external reverse proxy replacing its internal components. Traefik proxies it at the TCP layer (SNI passthrough or HTTP proxy). The "no nginx/apache" rule applies to all other services - GitLab is the explicit exception per your instructions.

---

### Layer 3 - Collaboration (Nextcloud)

| VM name | Tool | vCPU | RAM | Disk | IP | Notes |
|---|---|---|---|---|---|---|
| vm-nextcloud-poc-01 | Nextcloud | 2 | 2 GB | 20 GB | 10.1.3.35 | Docker (nextcloud-fpm + nginx sidecar + PostgreSQL + Redis). Primary storage: Contabo S3. SSO via Authentik (OIDC). `nextcloud.by-systems.be`. |

**Storage layout for Nextcloud:**
- User files → Contabo S3 (primary storage, unlimited scale within 250 GB quota)
- NAS via External Storage app: deferred - Nextcloud VM does not mount NFS at PoC stage. Revisit if NAS browsing is needed.

**nginx sidecar note:** Nextcloud-fpm requires a web server sidecar for PHP-FPM. This is a Nextcloud-internal component (fpm process manager), not a general-purpose reverse proxy. Traefik remains the external entry point. This is not a violation of the no-nginx rule - same pattern as GitLab's bundled components.

---

### Layer 4 - IPAM & DCIM (NetBox)

| VM name | Tool | vCPU | RAM | Disk | IP | Notes |
|---|---|---|---|---|---|---|
| vm-netbox-poc-01 | NetBox | 2 | 2 GB | 20 GB | 10.1.3.30 | Docker (netbox + PostgreSQL + Redis + netbox-worker). |

---

### Layer 5 - Package Registry (Nexus)

| VM name | Tool | vCPU | RAM | Disk | IP | Notes |
|---|---|---|---|---|---|---|
| vm-nexus-poc-01 | Nexus OSS | 2 | 2 GB | 50 GB | 10.1.3.40 | Docker. Stores pip, npm, Docker images, Maven. Large disk - artifacts accumulate. |

---

### Layer 6 - Observability (Prometheus + Grafana + Loki)

Colocated on one VM for PoC (separate later in prod).

| VM name | Tool | vCPU | RAM | Disk | IP | Notes |
|---|---|---|---|---|---|---|
| vm-observability-poc-01 | Prometheus + Grafana + Loki | 2 | 4 GB | 30 GB | 10.1.3.50 | Docker Compose. All three colocated - acceptable for PoC. Prometheus scrapes all VMs. Loki S3 backend (Contabo) offloads chunks; 4 GB covers query path with 13 VMs shipping logs. |

---

### Layer 7 - DNS, VPN & Network Management

| VM name | Tool | vCPU | RAM | Disk | IP | Notes |
|---|---|---|---|---|---|---|
| vm-pihole-poc-01 | Pi-hole | 1 | 512 MB | 10 GB | 10.1.1.60 | Docker. Pi-hole blocklist + local DNS overrides (`*.by-systems.be`). **No Unbound sidecar.** Upstream = OPNsense Unbound :853. Provisioned via Pi-hole v6 REST API (Ansible). OPNsense NAT redirects all :53 to this VM. |
| ~~vm-wireguard-poc-01~~ | ~~WireGuard~~ | - | - | - | - | **Removed.** WireGuard runs as OPNsense built-in plugin (`os-wireguard`). No separate VM needed. |
| vm-unifi-poc-01 | Unifi Network App | 1 | 1 GB | 10 GB | 10.1.1.62 | Docker. Ubiquiti controller - manages PoC WiFi AP + VLAN config. Mirrors prod Ubiquiti setup. |

**DNS architecture (authoritative):**

```
VMs / clients
    ↓ :53
OPNsense NAT redirect (intercepts all :53 - blocks bypass even if VM hardcodes 8.8.8.8)
    ↓ :53
Pi-hole (10.1.1.60) - blocklists + local DNS overrides (*.by-systems.be → private IPs)
    ↓ upstream :853
OPNsense Unbound (DoT terminator)
    ↓ DoT :853
1.1.1.1 / 1.0.0.1 (Cloudflare)
```

- **OPNsense firewall NAT rule:** redirect all DNS :53 from PoC zones → Pi-hole `10.1.1.60:53`. Enforced at FW - no VM can bypass Pi-hole.
- **Pi-hole:** blocklists + local custom DNS records only. **No Unbound sidecar** - upstream is OPNsense internal IP :853.
- **OPNsense Unbound:** enabled as DoT terminator. Forwards external queries to `1.1.1.1:853` + `1.0.0.1:853`. Not used directly by VMs.
- **OPNsense DHCP** pushes Pi-hole IP as DNS server to DHCP clients (belt + suspenders alongside NAT redirect).
- **Local DNS overrides:** `*.by-systems.be` → internal IPs configured via Pi-hole v6 REST API (Ansible-managed, no manual UI).
- **Cert issuance (DNS-01):** Traefik → Cloudflare API → public TXT record → Let's Encrypt validates. Pi-hole not involved.
- **Monitoring:** Pi-hole query log + dashboard. Grafana alert on Pi-hole health (down = DNS blind).
- **SPOF mitigation (PoC):** if Pi-hole VM fails, remove OPNsense NAT redirect → OPNsense Unbound serves DNS directly as fallback.

**OPNsense:** deployed first - virtual router + firewall + WireGuard server (`os-wireguard` plugin). No WireGuard VM needed. Rune VM and human clients connect as WireGuard peers. Migration: change endpoint only.

---

### Deferred (not in PoC scope)

| Tool | Reason |
|---|---|
| step-ca | Deferred - LE via Cloudflare DNS-01 covers PoC. step-ca for internal mTLS later. CT log exposure acceptable for PoC (wildcard cert hides subdomains). |
| DNS → BIND9 | **Deferred to Phase 2.** Pi-hole covers PoC (blocklists + local overrides). Phase 2: replace with BIND9 - authoritative zones, split-horizon views, TSIG, DNSSEC. CISO-grade DNS. `vm-pihole-poc-01` → `vm-bind-poc-01`. Zone files versioned in git, Ansible-managed. |
| Wazuh | **Deferred to Phase 2.** stack.md lists as CISO Tier 1. PoC proceeds without SIEM - acceptable risk for internal-only PoC. Must be first service added in Phase 2. |
| Teleport | **Deferred to Phase 2.** stack.md designates as primary bastion (cert SSH + session recording). SSH is direct to VMs for PoC. Acceptable for controlled single-operator environment. Add before multi-operator phase. |
| MinIO | **Removed from PoC.** Contabo S3 (250 GB) is available - use directly. No self-hosted S3 needed. |
| Zabbix | **Deferred.** Prometheus exporters cover PoC. SNMP revisit in Phase 3. |
| OPNsense | **Promoted to Layer 0** - deploys first, network foundation. |
| pfSense | Physical/VM firewall - outside Terraform VM scope. |
| OpenClaw | Already running on Rune VM. |

---

## Total PoC resource estimate

| Metric | Total |
|---|---|
| VMs | 13 | - | - |
| vCPU total | 20 vCPU | 12 physical cores | Overcommit ~1.7x - fine, workloads are I/O-bound |
| RAM total | ~27 GB | **188 GB** | **161 GB free - no constraints at all** |

*Sizing bumps from Opus audit (2026-04-01): Traefik 512 MB → 1 GB, GitLab 4 GB → 8 GB, Observability 2 GB → 4 GB.*
| Disk (VM OS) | ~260 GB | Proxmox local (thin) | - |
| Persistent data | Contabo S3 | 250 GB S3 | - |

**CPU:** 20 vCPU across 12 physical cores - overcommit ~1.7x. Fine for these workloads (mostly idle services, I/O-bound not CPU-bound). No pinning needed for PoC.
**RAM:** 21 GB allocated out of 188 GB physical = **11% used. No constraints.** Minimum sizes set now - trivial to bump any VM later without redesign.
**Disk:** VM OS disks on `poc-data` ZFS pool. Persistent app data on Contabo S3 (not NAS). ZFS gives native snapshotting - use for VM backups before major changes.

---

## Storage strategy

| Data type | Where | How |
|---|---|---|
| VM OS disks | **`poc-data` ZFS pool** (Proxmox local) | Terraform `datastore_id = "poc-data"`. ZFS, not LVM-thin. |
| Persistent app data | **Contabo S3** | Docker volumes → S3. All runtime data. No VM-level NFS. |
| GitLab repos | **`poc-data` ZFS** (VM local disk, 50 GB) | Git data lives on VM disk. Large repos scale to S3 object storage. |
| GitLab CI artifacts / backups | `by-poc-gitlab` (Contabo S3) | GitLab object storage config (S3-compatible) |
| Nexus blob store | `by-poc-nexus` (Contabo S3) | Nexus S3 blob store config |
| Loki log chunks | `by-poc-loki` (Contabo S3) | Loki S3 backend |
| Nextcloud user files | `by-poc-nextcloud` (Contabo S3) | Nextcloud primary storage backend |
| Prometheus TSDB | Local VM disk (fast I/O) | Keep on VM - latency sensitive |
| General backups | `by-poc-backups` (Contabo S3) | Restic - versioning enabled |
| Proxmox ISOs | **NAS NFS host-level** (`poc-iso`) | Proxmox storage config - host mounts NFS, VMs don't see it |
| Proxmox VM backups | **NAS NFS host-level** (`poc-backup`) | Proxmox backup jobs - host-level NFS mount, not VM traffic |

---

## Docker + Traefik deployment rules

1. **Every service** runs in Docker (exception: GitLab bundled components)
2. **Traefik** is the single HTTP/S entry point - all services expose only to Traefik network, not to host
3. **Traefik labels** on each Docker container define routing (`traefik.http.routers.*`)
4. **TLS:** Let's Encrypt via Traefik + **Cloudflare DNS-01 challenge**. Real domain `by-systems.be` - subdomains like `gitlab.by-systems.be`, `vault.by-systems.be`. Traefik handles cert issuance + renewal automatically. Cloudflare token stored in Vault, injected into Traefik at runtime. Works for internal services (DNS-01 does not require public HTTP access). step-ca is deferred - it handles internal mTLS between services, not browser-facing certs.
5. **Docker networks:** one `traefik-net` bridge network shared across all services + Traefik
6. **No port 80/443 on individual VMs** - only Traefik VM exposes 80/443 to the network
7. **Compose files** live in `platform-setup/tools/{tool}/docker/` - tracked in git, no manual docker run commands

---

## Cluster path (for reference - not PoC scope)

```
Now:       Docker Compose per VM (single host)
Phase 2:   Docker Swarm (multi-host, same Docker tooling)
Phase 3:   K3s (lightweight Kubernetes, when Swarm limits hit)
```

Compose files written with Swarm compatibility in mind (`deploy:` blocks commented but present).

---

## Pre-deployment checklist (before Terraform runs)

- [ ] Debian 12 cloud-init template exists on Proxmox (confirm VMID + stored on `poc-data`)
- [ ] `poc-data` ZFS pool confirmed healthy (`zpool status poc-data` on Proxmox)
- [ ] Terraform `datastore_id = "poc-data"` set in all VM modules (not `local-lvm`)
- [ ] NFS exports on NAS configured for Proxmox host IP (10.6.224.105): `poc-iso` + `poc-backup` shares. No PoC VLAN export needed - host-level mounts only.
- [ ] Proxmox SDN configured: `vnet-mgmt`, `vnet-dmz` VNets created
- [ ] Rune VM second NIC added to `vnet-mgmt` (`10.1.1.5`)
- [ ] IP plan confirmed (no conflicts with DHCP pool `10.6.239.101-199`)
- [ ] Terraform module `vm-linux` tested (bootstrap VM on `poc-data`)
- [ ] `svc-terraform-poc` credentials active on PoC Proxmox
- [ ] Cloudflare token key name confirmed in `infra/secrets/`
- [ ] Contabo S3 day0 Terraform run complete (5 buckets created: by-poc-gitlab, by-poc-nexus, by-poc-loki, by-poc-nextcloud, by-poc-backups)

---

## Deployment order

```
0.  vm-opnsense-poc-01        (firewall/SDN - DEPLOY FIRST, network foundation)
1.  vm-traefik-poc-01         (entry point - DMZ zone, HTTP proxy mode for GitLab)
2.  vm-pihole-poc-01          (DNS - needed for service discovery)
3.  vm-netbox-poc-01          (IPAM - source of truth, register all IPs before further deploys)
4.  vm-vault-poc-01           (secrets - needed by Authentik)
5.  vm-vaultwarden-poc-01     (human credentials - standalone, deploy alongside Vault)
6.  vm-authentik-poc-01       (identity - needed by all services; ADR-0004 covers identity federation architecture)
7.  vm-gitlab-poc-01          (VCS - needed by CI)
8.  vm-gitlab-runner-poc-01   (CI - depends on GitLab)
9.  vm-nexus-poc-01           (registry - standalone)
10. vm-observability-poc-01   (observability - scrapes all above)
11. vm-nextcloud-poc-01       (collaboration - after Authentik, SSO required)
12. vm-unifi-poc-01           (network mgmt - WiFi AP + VLAN)
```

**Removed:** ~~vm-wireguard-poc-01~~ - WireGuard runs on OPNsense (`os-wireguard` plugin), no separate VM needed.

---

## Confirmed

| Item | Decision |
|---|---|
| TLS | Let's Encrypt via Traefik + Cloudflare DNS-01. Domain: `*.by-systems.be`. Token in Vault. Cloudflare token rotation: document runbook + Grafana cert-expiry alert before PoC go-live. |
| GitLab TLS | HTTP proxy mode: Traefik terminates TLS, forwards HTTP to GitLab:8080. SNI passthrough rejected. |
| S3 object storage | Contabo S3 (250 GB) - GitLab artifacts, Nexus blobs, Loki chunks, backups. No MinIO VM. |
| step-ca | Deferred - LE covers PoC. |

## Confirmed

| Item | Decision |
|---|---|
| RAM | **188 GB - no constraints.** |
| NFS shares | `poc-iso` + `poc-backup` - existing on NAS. |
| DNS (Phase 1 - PoC) | Pi-hole (10.1.1.60) - blocklists + local overrides. OPNsense NAT enforces :53 redirect. OPNsense Unbound = DoT terminator :853. **Phase 2 migration: replace Pi-hole → BIND9** (authoritative, split-horizon, TSIG, DNSSEC - CISO-grade). Pi-hole is PoC-only. |
| Domain | `{service}.by-systems.be` confirmed. |
| Cloudflare token | In `infra/secrets/` - confirm key name before Traefik deploy. |
| Contabo S3 creds | In `infra/secrets/` - confirm bucket names before GitLab/Nexus/Loki deploy. |
| Zabbix | Deferred. |
| Unifi | `vm-unifi-poc-01` added. 1x WiFi AP in PoC + VLAN via OPNsense (when deployed). |
| step-ca | Deferred. |
| MinIO | Removed - Contabo S3 used directly. |

## ✅ All questions resolved

| Item | Answer |
|---|---|
| NAS volume | `/volume1/poc-iso` + `/volume1/poc-backup` (9TB global volume) |
| Contabo S3 buckets | `by-poc-gitlab`, `by-poc-nexus`, `by-poc-loki`, `by-poc-nextcloud`, `by-poc-backups` |
| Bucket creation | Terraform `day0` module (`aws` provider → Contabo S3 endpoint). Runs before any service deploy. |
| Bucket versioning | Enabled on `by-poc-gitlab` + `by-poc-backups`. All others default. |
| Public access | Blocked on all buckets. |

---

*Rune proposes. Opus validates BoM against layer model. @yboujraf approves. Then Terraform.*
