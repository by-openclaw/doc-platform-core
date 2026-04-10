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

| Range | Zone | Purpose |
|---|---|---|
| 1-2000 | prod | All production segments |
| 2001-4094 | test | All non-prod (dev, test, staging, acc) |

### 3. Production VLAN Assignment

| VLAN ID | Segment | Subnet (v4) | Subnet (v6 ULA) | Gateway |
|---|---|---|---|---|
| 100 | OOB | 10.0.100.0/24 | — | — (L2 only, no routing) |
| 200 | FABRICS | 10.0.200.0/24 | — | — (isolated) |
| 300 | MGMT | 10.1.1.0/24 | fd00:1:1::/64 | 10.1.1.1 |
| 310 | DMZ | 10.1.2.0/24 | fd00:1:2::/64 | 10.1.2.1 |
| 320 | SVC | 10.1.3.0/24 | fd00:1:3::/64 | 10.1.3.1 |
| 330 | VPN | 10.1.4.0/24 | fd00:1:4::/64 | 10.1.4.1 |
| 400 | IoT | 10.1.10.0/24 | fd00:1:10::/64 | 10.1.10.1 |
| 410 | VoIP | 10.1.11.0/24 | fd00:1:11::/64 | 10.1.11.1 |
| 500 | Storage | 10.1.20.0/24 | — | 10.1.20.1 |
| 600 | Media | 10.1.30.0/24 | fd00:1:30::/64 | 10.1.30.1 |
| 700 | CCTV | 10.1.40.0/24 | — | 10.1.40.1 |

### 4. Test VLAN Assignment (offset +2000)

| VLAN ID | Segment | Subnet (v4) | Subnet (v6 ULA) | Gateway |
|---|---|---|---|---|
| 2300 | MGMT | 10.11.1.0/24 | fd11:1::/64 | 10.11.1.1 |
| 2310 | DMZ | 10.11.2.0/24 | fd11:2::/64 | 10.11.2.1 |
| 2320 | SVC | 10.11.3.0/24 | fd11:3::/64 | 10.11.3.1 |
| 2330 | VPN | 10.11.4.0/24 | fd11:4::/64 | 10.11.4.1 |

Test zone only creates segments needed for lib-opnsense integration testing.
Additional test segments (IoT, VoIP, etc.) added when needed.

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
| test | Test/Dev/Staging/Acc | tmgmt, tdmz, tsvc, tvpn |

### 8. Merges Applied

| Original items | Merged into | Reason |
|---|---|---|
| APPS + SVC | SVC | Same purpose — internal services |
| SDN | Not a segment | SDN is the trunk mechanism, not a VLAN |
| TV + IPTV + OTT + gaming | Media | All entertainment traffic, same trust |
| Printer + IoT | IoT | Both untrusted devices, same security posture |
| VPN + VPN inter-site | VPN | Same FW rules, different tunnels |

## Consequences

- ADR-0015 VLAN IDs (310/320/330) change to match new registry — migration needed
- Current test VLANs (1310-1340) need migration to 2300+ range — planned with vm-fw-test-01 (1100)
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
