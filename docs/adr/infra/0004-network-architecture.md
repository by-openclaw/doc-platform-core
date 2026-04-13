# infra/0004 — Network Architecture

**Status:** Draft
**Date:** 2026-04-13 (supersedes flat ADR-0015 + flat ADR-0032, merging the architectural content of both)
**Scope:** Network zones, VLAN registry, Proxmox SDN architecture, WireGuard VPN, dual-stack IPv4 + IPv6. Does not define OPNsense firewall rules, bridge/WAN operational state, or physical cabling.
**Related:** `naming/0001-infra`, `naming/0003-firewall`, `infra/0005-environment-tiers`, `security/0004-certificate-strategy`, `services/0001-opnsense-provisioning` (future)

---

## Context

The platform requires a scalable, reproducible network topology. Earlier attempts (flat ADR-0015) defined a narrow PoC-only VLAN set with no taxonomy for how segments generalize across environments. Flat ADR-0032 supersedes 0015's VLAN assignments with an 11-segment model and VLAN range allocation that scales to production. This ADR merges the architectural content of both into one authoritative source.

**Current operational state** (bridges, WAN uplinks, physical wiring, current deployment phase) is **not** in this ADR — it belongs in NetBox + runbooks. This ADR captures the **architectural rules** that are invariant across current and future deployments.

## Decision

### 1. Eleven network segments — zone taxonomy

| # | Segment | Purpose | Trust | Inter-VLAN | Internet | Notes |
|---|---|---|---|---|---|---|
| 1 | **OOB** | iLO, iDRAC, switch console, break-glass | Highest | No | No | Never routed through firewall |
| 2 | **MGMT** | Platform admin, monitoring, Terraform, Ansible | High | → SVC, DMZ | Yes (updates) | Management plane |
| 3 | **DMZ** | Public-facing services (Traefik, web) | Low | ← WAN | Yes (in+out) | Exposed to WAN |
| 4 | **SVC** | Internal platform services + apps | Medium | → DMZ | Yes (outbound) | Not directly exposed |
| 5 | **FABRICS** | Switch fabric control plane (Arista) | High | VRF leak | No | Routed via VRF on Arista |
| 6 | **VPN** | Remote access (WireGuard / OpenVPN / IPsec) + site-to-site | Medium | Per user | Via tunnel | Firewall rules per tunnel |
| 7 | **IoT** | Sensors, printers, smart devices | Untrusted | No | Restricted | No access to MGMT / SVC |
| 8 | **VoIP** | Phone system (SIP, RTP) | Medium | No | Yes (SIP trunk) | QoS priority |
| 9 | **Storage** | Ceph, NAS, backup traffic | High | → SVC only | No | Data plane only |
| 10 | **Media** | TV, IPTV, OTT streaming, gaming | Untrusted | No | Yes (streaming) | No access to internal |
| 11 | **CCTV** | Security cameras, NVR | Isolated | No | No | Air-gapped |

**Rules:**
- All VLAN subnets are **/24 minimum** — no smaller masks
- OOB and FABRICS are **not** VLANs on Proxmox SDN — they are dedicated Proxmox bridges (`vmbrOOB`, `vmbrFAB`) because they must remain reachable even if SDN is reconfigured
- Trust levels are enforced by OPNsense firewall rules — see `naming/0003-firewall` for rule description format

### 2. VLAN range allocation

| Range | Purpose | Notes |
|---|---|---|
| **1–999** | Reserved — ISP, Arista fabric, broadcast | Do NOT allocate |
| **1001–1999** | **Prod** — all production segments | BY-SYSTEMS platform infra |
| **2001–2999** | **Test** — all non-prod (dev, test, staging, acc) | Integration testing |
| **3001–4094** | Future — additional envs if needed | Unallocated |

**Reserved VLANs (1–999) — do not use:**

| VLAN | Owner | Purpose |
|---|---|---|
| 1 | Never use | VLAN hopping risk (802.1Q double-tagging) |
| 10 | Proximus | PPPoE WAN uplink |
| 20 | Proximus | Reserved |
| 110–121 | Arista | P2P fabric links (7060 / 7020) |
| 600–999 | Arista / ISP | Broadcast control (SMPTE, PTP, MGMT_CTRL), Telenet (999) |

### 3. Production VLAN assignment (1001–1999)

Dual-stack IPv4 + IPv6 on every segment. IPv6 uses ULA (fd00::/8) for internal addressing.

| VLAN ID | Segment | IPv4 subnet | IPv6 ULA | Gateway |
|---|---|---|---|---|
| 1010 | MGMT | 10.1.1.0/24 | fd01:1::/64 | 10.1.1.1 |
| 1020 | DMZ | 10.1.2.0/24 | fd01:2::/64 | 10.1.2.1 |
| 1030 | SVC | 10.1.3.0/24 | fd01:3::/64 | 10.1.3.1 |
| 1040 | VPN | 10.1.4.0/24 | fd01:4::/64 | 10.1.4.1 |
| 1100 | IoT | 10.1.10.0/24 | fd01:10::/64 | 10.1.10.1 |
| 1110 | VoIP | 10.1.11.0/24 | fd01:11::/64 | 10.1.11.1 |
| 1200 | Storage | 10.1.20.0/24 | fd01:20::/64 | 10.1.20.1 |
| 1300 | Media | 10.1.30.0/24 | fd01:30::/64 | 10.1.30.1 |
| 1400 | CCTV | 10.1.40.0/24 | fd01:40::/64 | 10.1.40.1 |

### 4. Test VLAN assignment (2001–2999)

Test range uses `+1000` offset from prod and `10.11.x.x` / `fd11:x::/64` addressing — mnemonic: test = prod with a leading `1` on the second octet.

| VLAN ID | Segment | IPv4 subnet | IPv6 ULA | Gateway |
|---|---|---|---|---|
| 2010 | MGMT | 10.11.1.0/24 | fd11:1::/64 | 10.11.1.1 |
| 2020 | DMZ | 10.11.2.0/24 | fd11:2::/64 | 10.11.2.1 |
| 2030 | SVC | 10.11.3.0/24 | fd11:3::/64 | 10.11.3.1 |
| 2040 | VPN | 10.11.4.0/24 | fd11:4::/64 | 10.11.4.1 |
| 2100 | IoT | 10.11.10.0/24 | fd11:10::/64 | 10.11.10.1 |
| 2110 | VoIP | 10.11.11.0/24 | fd11:11::/64 | 10.11.11.1 |
| 2200 | Storage | 10.11.20.0/24 | fd11:20::/64 | 10.11.20.1 |
| 2300 | Media | 10.11.30.0/24 | fd11:30::/64 | 10.11.30.1 |
| 2400 | CCTV | 10.11.40.0/24 | fd11:40::/64 | 10.11.40.1 |

The test range covers `dev`, `test`, `staging`, and `acc` env tiers. Higher tiers (`prod`, `drp`) use the production range. Env tier definitions are in `infra/0005-environment-tiers`.

### 5. VMID scheme

Proxmox VMIDs are allocated per zone and per VM vs LXC, to make ownership visible at a glance in the Proxmox UI.

| Zone | VM range | LXC range |
|---|---|---|
| **prod** | 100–499 | 500–999 |
| **test** | 1100–1499 | 1500–1999 |

### 6. Proxmox SDN architecture

Proxmox SDN provides L2 switching only. **OPNsense is the only router.**

| Layer | Responsibility |
|---|---|
| **Proxmox SDN Zone** (VLAN type) | L2 switching within Proxmox — VMs on the same VNet talk directly, no gateway needed |
| **Proxmox SDN VNet** | Creates a VLAN tag on the physical trunk bridge; appears as a local interface on each Proxmox node |
| **Proxmox SDN Subnet** | Optional metadata (IPAM reference only — Proxmox creates no gateway) |
| **OPNsense** | Gateway, inter-VLAN routing, DHCP, DNS, firewall, IPv4+IPv6 — the ONLY router on the platform |

VMs on different VNets **must** traverse OPNsense, and OPNsense firewall rules decide whether the traffic is allowed — see `naming/0003-firewall`.

### 7. SDN zone naming

Zone name = environment tier (per `infra/0005-environment-tiers`). VNet names are segment abbreviations.

| Zone | Environment tiers | VNets |
|---|---|---|
| `prod` | `prod`, `drp` | `mgmt`, `dmz`, `svc`, `vpn`, `iot`, `voip`, `storage`, `media`, `cctv` |
| `test` | `dev`, `test`, `staging`, `acc` | `tmgmt`, `tdmz`, `tsvc`, `tvpn`, `tiot`, `tvoip`, `tstor`, `tmedia`, `tcctv` |

**Rules:**
- Zone name is env-tier-bound — one zone per tier group, not per individual tier (test zone serves dev/test/staging/acc; prod zone serves prod/drp)
- VNet names are **environment-agnostic within their zone** — the env context lives in the VM name and NetBox record, not in the VNet name
- Adding a new Proxmox cluster reuses the same zone names — zone names are portable across clusters

### 8. WireGuard VPN

WireGuard runs as a virtual interface on OPNsense. **It does not require a dedicated VLAN** — tunnels are routed by OPNsense and subject to firewall rules.

| Item | Value |
|---|---|
| Endpoint | OPNsense WAN interface, UDP **51820** |
| Tunnel subnet (prod) | `10.1.4.0/24` (VPN segment, VLAN 1040) |
| Tunnel subnet (test) | `10.11.4.0/24` (VPN segment, VLAN 2040) |
| OPNsense interface IP (prod) | `10.1.4.1/24` |
| DMZ anchor IP | OPNsense DMZ-side interface in the respective env (WireGuard listens here) |

#### Peer naming

One peer **per device**, never per user. A user with a phone and a laptop has two peers.

```
peer-{owner}-{device}
```

| Peer name | Owner | Device | Tunnel IP |
|---|---|---|---|
| `peer-yboujraf-win11` | yboujraf | Win11 workstation | `10.1.4.2/32` |
| `peer-yboujraf-mobile` | yboujraf | Mobile | `10.1.4.3/32` |
| `peer-rune-vm` | automation | Rune VM | `10.1.4.4/32` |

**Rules:**
- Adding a new device = add one peer entry — no firewall rule changes (the `vpn` zone is the firewall rule subject)
- **Peer keypairs are generated per device.** Private keys never leave the device. The OPNsense-side registration holds only the public key.
- **No device = no peer entry.** If the device is lost or decommissioned, the peer is removed immediately.

#### Access from VPN

WireGuard peers land in the `VPN` segment (10.1.4.0/24 prod, 10.11.4.0/24 test). From there, OPNsense firewall rules decide what is reachable:

- **Default allow:** VPN → MGMT (`mgmt` VNet), VPN → SVC (`svc` VNet)
- **Default deny:** VPN → DMZ (public path is via the internet, not through the VPN)
- **Default deny:** VPN → IoT, Storage, Media, CCTV (untrusted or isolated zones)

#### Secret storage

WireGuard server private key is stored in HashiCorp Vault at `secret/{env}/opnsense/wireguard-server-key` per `security/0001-secret-storage`. Peer public keys are deployed via Ansible (`ansible-opnsense`) and version-controlled in git (public keys only).

### 9. Dual-stack IPv4 + IPv6 — mandatory

Every VLAN carries both an IPv4 subnet and an IPv6 ULA prefix. Every host on that VLAN receives both an IPv4 address and an IPv6 address. **Single-stack deployment is not allowed** (consistent with `naming/0001-infra §6.1`).

Exceptions:
- **IoT / Media / CCTV** segments may be IPv4-only if the devices do not support IPv6, but the VLAN itself still has IPv6 provisioned in case supported devices join later.
- **OOB** is IPv4-only (management plane, no IPv6 requirement).

### 10. Cross-OS note

Network architecture is OS-agnostic. Linux hosts and future Windows 11 / Windows Server hosts (see `git/0003-configuration §11`) share the same VLANs, VNets, and firewall rules. Per-OS config differences (DHCP client, firewall, WireGuard client) are operational, not architectural, and live in the Ansible roles for each OS.

## Consequences

- **VLAN 340 / 350** from the original flat ADR-0015 are permanently retired — must not appear in switch configs, Proxmox SDN zones, or OPNsense interfaces
- **10.6.225.x** range from flat 0015 was a temporary bootstrap range — must not be used as canonical addressing; all new provisioning uses the 10.1.x.x (prod) or 10.11.x.x (test) runtime addressing
- **Zone `poc` is removed** — replaced by `prod` and `test` zones per `infra/0005-environment-tiers`
- **Proxmox clustering** is supported by design — all zones, VNets, VLANs, and subnets replicate across cluster nodes unchanged
- **Bridges, WAN uplinks, and physical wiring** are explicitly NOT in this ADR — they are operational state tracked in NetBox and per-node runbooks
- **Firewall alias standard** and OPNsense rule naming live in `naming/0003-firewall` — this ADR references the rules but does not duplicate them
- **OPNsense is a single point of routing** — it is the only router; losing it partitions the network. High availability for OPNsense is a Layer 2 prerequisite per `infra/0002-platform-charter`.

## Revision triggers

Revise when:
- A 12th network segment is required
- The VLAN allocation ranges (1001–1999 prod, 2001–2999 test) are exhausted
- Proxmox SDN is replaced (e.g. by Cilium or kube-vxlan)
- OPNsense is replaced as the platform router
- IPv6 GUA (globally routed) is adopted alongside ULA (currently ULA only for internal, GUA via Cloudflare for public edge)
- A new physical site is added (adds a site code per `naming/0001-infra §4`, may require VLAN range extension)
- A second Proxmox cluster is added in a different site

## CISO mapping

| Framework | Controls covered |
|---|---|
| ISO 27001:2022 | A.8.20 (networks security — 11-segment model with trust levels), A.8.22 (segregation of networks — untrusted zones isolated), A.8.21 (security of network services — OPNsense firewall gate) |
| NIS2 | Art. 21(2)(a) (risk management — per-segment trust levels), Art. 21(2)(e) (network and information systems security — version-controlled VLAN registry) |
