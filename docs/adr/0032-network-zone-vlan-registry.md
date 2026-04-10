# ADR-0032: Network Zone & VLAN Registry

**Status:** Accepted
**Date:** 2026-04-10
**Deciders:** @yboujraf
**Supersedes:** ADR-0015 (Network VLAN Architecture) — zone taxonomy and VLAN ranges

## Context

ADR-0015 defined a PoC VLAN set (300-330) without a scalable zone taxonomy. The platform needs 11 network segments across production and test environments, with VLAN IDs constrained to 1-4094 (802.1Q, 12-bit). A `poc` zone was used but `poc` is not a valid environment tier (ADR-0012). The infrastructure must support future Proxmox clustering and multi-site VPN.

## Decision

### 1. Network Segments — 11 confirmed

| # | Segment | Purpose | Trust level | Internet | Notes |
|---|---|---|---|---|---|
| 1 | OOB | Hardware mgmt (iLO, iDRAC, switch console), break-glass | Highest | No | Never routed through FW |
| 2 | MGMT | Platform admin, monitoring, Proxmox API, TF, Ansible | High | Via FW rules | Management plane |
| 3 | DMZ | Public-facing services (Traefik, web) | Low | Yes (inbound + outbound) | Exposed to WAN |
| 4 | SVC | Internal platform services + applications | Medium | Outbound only via FW | Not directly exposed |
| 5 | FABRICS | Switch fabric control plane (Arista 7060/7020) | High | No | Isolated, vmbrFAB |
| 6 | VPN | Remote access (WireGuard, OpenVPN, IPsec) + site-to-site | Medium | Via tunnel only | FW rules per tunnel |
| 7 | IoT | Sensors, printers, smart devices | Untrusted | Restricted | No access to MGMT/SVC |
| 8 | VoIP | Phone system (SIP, RTP) | Medium | SIP trunk only | QoS priority |
| 9 | Storage | Ceph, NAS, backup traffic | High | No | Isolated, high bandwidth |
| 10 | Media | TV, IPTV, OTT streaming, gaming | Untrusted | Yes | No access to internal |
| 11 | CCTV | Security cameras, NVR | Isolated | No | Air-gapped from all other segments |

### 2. VLAN Range Allocation

| Range | Purpose | Notes |
|---|---|---|
| 1-999 | **Reserved** — ISP, Arista fabric, broadcast | Do NOT allocate |
| 1001-1999 | **Prod** — all production segments | BY-SYSTEMS infra |
| 2001-2999 | **Test** — all non-prod (dev, test, staging, acc) | Integration testing |
| 3001-4094 | **Future** — additional envs if needed | Unallocated |

### Reserved VLANs (1-999) — DO NOT USE

| VLAN | Owner | Purpose |
|---|---|---|
| 1 | Dead | Never use — VLAN hopping risk (802.1Q double-tagging) |
| 10 | Proximus | PPPoE WAN uplink |
| 20 | Proximus | Reserved |
| 110-121 | Arista | P2P fabric links (7060/7020) |
| 600-999 | Arista/ISP | Broadcast control (SMPTE, PTP, MGMT_CTRL), Telenet (999) |

### 3. Production VLAN Assignment (1001-1999)

| VLAN ID | Segment | Subnet (v4) | Subnet (v6 ULA) | Gateway |
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

OOB and FABRICS are NOT VLANs on SDN — they are dedicated Proxmox bridges (vmbrOOB, vmbrFAB).

### 4. Test VLAN Assignment (2001-2999, offset +1000 from prod)

All segments dual-stack (IPv4 + IPv6 ULA).

| VLAN ID | Segment | Subnet (v4) | Subnet (v6 ULA) | Gateway |
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

### Migration from current test VLANs

| Current | New | Segment |
|---|---|---|
| 1310 | 2010 | MGMT |
| 1320 | 2020 | DMZ |
| 1330 | 2030 | SVC |
| 1340 | 2040 | VPN |

Applied with VM 1100. VM 101 stays on 1310-1340 until decommissioned.

### 5. VMID Scheme

| Zone | VM range | LXC range |
|---|---|---|
| prod | 100-499 | 500-999 |
| test | 1100-1499 | 1500-1999 |

### 6. Proxmox Bridges

| Bridge | Status | Purpose |
|---|---|---|
| vmbrOOB | Active | Hardware OOB management (iLO, switches). No VMs. Break-glass only. |
| vmbrWAN1 | Active | WAN1 Proximus PPPoE |
| vmbrWAN2 | Active | WAN2 Telenet |
| vmbrWAN3 | Active | Temporary internet via pfSense OOB (10.6.224.0/20) |
| vmbrFAB | Disabled | Fabric control plane (Arista). Disabled until wired. |
| vmbrAPPS | Active | VLAN trunk. SDN zones attach here. OPNsense LAN. |

### 7. SDN Zone Naming

Zone name = environment tier (ADR-0012). VNet names are segment abbreviations.

| Zone | Environment | VNets |
|---|---|---|
| prod | Production | mgmt, dmz, svc, vpn, iot, voip, storage, media, cctv |
| test | Test/Dev/Staging/Acc | tmgmt, tdmz, tsvc, tvpn, tiot, tvoip, tstor, tmedia, tcctv |

### 8. Merges Applied

| Original items | Merged into | Reason |
|---|---|---|
| APPS + SVC | SVC | Same purpose — internal services |
| SDN | Not a segment | SDN is the trunk mechanism, not a VLAN |
| TV + IPTV + OTT + gaming | Media | All entertainment traffic, same trust |
| Printer + IoT | IoT | Both untrusted devices, same security posture |
| VPN + VPN inter-site | VPN | Same FW rules, different tunnels |

## Consequences

- ADR-0015 VLAN IDs (310/320/330) superseded — prod now starts at 1001
- Current test VLANs (1310-1340) migrate to 2010-2040 with vm-fw-test-01 (1100)
- VLANs 1-999 permanently reserved for ISP (10, 20), Arista fabric (110-121, 600-999)
- Zone `poc` is removed — replaced by `prod` (ADR-0012 compliance)
- All Terraform SDN configs updated to match new VLAN IDs
- Future segments added by appending to the registry — no structural change needed
- Proxmox cluster: same zones, same VLANs, replicated across nodes

## CISO mapping

### ISO/IEC 27001:2022

| Control | Title | Status | Notes |
|---|---|---|---|
| A.8.20 | Networks security | Covered | 11 segments with trust levels defined |
| A.8.22 | Segregation of networks | Covered | Untrusted (IoT, Media, CCTV) isolated from internal |

### NIS2 (Directive 2022/2555)

| Article | Requirement | Status | Notes |
|---|---|---|---|
| Art. 21(2)(a) | Risk management — network controls | Covered | Per-segment trust levels and internet access rules |
| Art. 21(2)(e) | Security in network and information systems | Covered | VLAN registry is version-controlled, auditable |

## References

- ADR-0012: Environment Tier Standard
- ADR-0015: Network VLAN Architecture (superseded for zone/VLAN ranges)
- ADR-0010: Naming & Identity Convention
