# Network Architecture — End-to-End ASCII Diagram

> **Companion to**: `adr/infra/0004-network-architecture.md`
> **Scope**: Two ISPs (Proximus PPPoE + Telenet static), HPE V1910-48G switch, Proxmox + OPNsense FW, SDN trunk + Fabric overlay.
> **Status**: as-built for `srv-proxmox-poc-01` + `vm-opns-test-01` (test env, 2026-05-23). Prod follows same pattern.

---

## 1. End-to-end topology

```
                            ╔═══════════════╗
                            ║   INTERNET    ║
                            ╚═══════════════╝
                                    │
              ┌─────────────────────┼─────────────────────┐
              │                                           │
       ┌──────┴──────────┐                       ┌────────┴────────┐
       │ ISP 1: PROXIMUS │                       │ ISP 2: TELENET  │
       │ Corporate Fiber │                       │ Corporate Fiber │
       │   PPPoE dyn     │                       │   Static IPs    │
       │   IPv4 dyn      │                       │   /29 IPv4      │
       │   /56 IPv6 PD   │                       │   /48 IPv6      │
       └────────┬────────┘                       └────────┬────────┘
                │ tagged VLAN 10                          │ tagged VLAN 999
                │ (PPPoE inside)                          │ (untagged inside)
                ▼                                         ▼
   ┌────────────────────────────────────────────────────────────────┐
   │                                                                │
   │            HPE V1910-48G SWITCH                                │
   │            (managed L2/L3 switch — VLAN trunking)              │
   │                                                                │
   │   Port 1   ← Proximus uplink   (tagged VLAN 10)                │
   │   Port 2   ← Telenet uplink    (tagged VLAN 999)               │
   │   Port 3-4 ← Proxmox POC trunk (tagged 10, 999, 600, OOB)      │
   │   Port 5-6 ← Proxmox PROD trunk(tagged 10, 999, 600, OOB)      │
   │   Port 7   ← NAS (Synology)    (untagged OOB or tagged 2200)   │
   │   Port 8…  ← Office Wi-Fi APs, admin workstations              │
   │                                                                │
   │   VLAN configuration:                                          │
   │     10   = WAN1 Proximus                                       │
   │     999  = WAN2 Telenet                                        │
   │     600  = FAB (Fabric supervision overlay)                    │
   │    OOB   = management network (untagged on some ports)         │
   │   1xxx   = PROD VLANs (1010=OPS, 1020=DMZ, …, 1400=CCTV)       │
   │   2xxx   = TEST VLANs (2010=OPS, 2020=DMZ, …, 2400=CCTV)       │
   │                                                                │
   └─────────────────────┬──────────────────────────────────────────┘
                         │ trunk (tagged 10, 999, 600, and SDN VLANs)
                         ▼
   ╔════════════════════════════════════════════════════════════════╗
   ║         PROXMOX HOST                                           ║
   ║         srv-proxmox-poc-01   (10.6.224.105 mgmt OOB)           ║
   ║                                                                ║
   ║  Physical NICs:                                                ║
   ║    nic0 → enp3s0f0  (port-1 to switch)                         ║
   ║    nic1 → enp3s0f1  (port-2 to switch)                         ║
   ║    nic2 → enp3s0f2                                             ║
   ║    nic3/nic4/nic5                                              ║
   ║    bond0  (LAG, OOB to office switch)                          ║
   ║                                                                ║
   ║  Kernel VLAN sub-interfaces (host-side VLAN tagging):          ║
   ║    nic0.10   ← VLAN 10  ← Proximus stripped here               ║
   ║    nic1.999  ← VLAN 999 ← Telenet stripped here                ║
   ║    nic4.600  ← VLAN 600 ← Fabric stripped here                 ║
   ║                                                                ║
   ║  Linux Bridges (vmbr*):                                        ║
   ║    vmbrOOB   port = bond0       (10.6.224.105/20 on host)      ║
   ║    vmbrAPPS  port = (none)      SDN trunk software-only,       ║
   ║                                  10.1.1.254/24 on host         ║
   ║    vmbrWAN1  port = nic0.10     ← Proximus untagged inside     ║
   ║    vmbrWAN2  port = nic1.999    ← Telenet untagged inside      ║
   ║    vmbrFAB   port = nic4.600    ← Fabric supervision           ║
   ║                                                                ║
   ║   ──────────────────────────────────────────────────────────   ║
   ║                                                                ║
   ║  VM 199 — vm-opns-test-01 (OPNsense FW)                        ║
   ║                                                                ║
   ║    Proxmox vNIC → Bridge        FreeBSD if  → OPNsense slot    ║
   ║    ───────────────────────────────────────────────────────     ║
   ║    net0          → vmbrAPPS  →  vtnet0      → OPT1 LAN_TRUNK   ║
   ║    net1          → vmbrOOB   →  vtnet1      → LAN (OOB_MGMT)   ║
   ║    net2          → vmbrWAN1  →  vtnet2      → WAN1 (pppoe0)    ║
   ║    net3          → vmbrWAN2  →  vtnet3      → WAN2 (static)    ║
   ║                                                                ║
   ╚══════════╤═════════════════════════════════════════════════════╝
              │ inside OPNsense FW
              ▼
   ╔════════════════════════════════════════════════════════════════╗
   ║                   OPNsense FW (vm-opns-test-01)                ║
   ║                                                                ║
   ║   ┌──────────────────────────────────────────────────────┐     ║
   ║   │ WAN1 = pppoe0 (parent vtnet2)                        │     ║
   ║   │   IPv4: PPPoE — Proximus dynamic                     │     ║
   ║   │   IPv6: DHCPv6 over PPP → /56 PD                     │     ║
   ║   │     PD example: 2a02:a03f:6098:9000::/56             │     ║
   ║   │   MTU: 1492 / MSS 1452                               │     ║
   ║   │   block_priv + block_bogons: ON                      │     ║
   ║   │                                                      │     ║
   ║   │ WAN2 = vtnet3                                        │     ║
   ║   │   IPv4: static 213.214.47.222/29 (gw .217)           │     ║
   ║   │   IPv6: static 2a02:1802:21::2/64 (gw ::1)           │     ║
   ║   │     + /48 routed: 2a02:1802:21::/48 → ::2            │     ║
   ║   │   block_priv + block_bogons: ON                      │     ║
   ║   │                                                      │     ║
   ║   │ LAN  = vtnet1                                        │     ║
   ║   │   IPv4: DHCP from OOB upstream (10.6.224.0/20)       │     ║
   ║   │   IPv6: track WAN1 PD / GUA from Telenet /48         │     ║
   ║   │   Anti-lockout: ON (admin always reachable from LAN) │     ║
   ║   │                                                      │     ║
   ║   │ OPT1 = vtnet0 = LAN_TRUNK                            │     ║
   ║   │   IPv4: 10.11.1.1/24    IPv6: …:0001::1/64           │     ║
   ║   │   VLAN parent (no clients directly)                  │     ║
   ║   └──────────────────────────────────────────────────────┘     ║
   ║                          │ trunk                               ║
   ║                          ▼ (VLAN sub-interfaces on vtnet0)     ║
   ║   ┌────────┬───────┬──────────────────┬──────────────────┐     ║
   ║   │ slot   │ VLAN  │  IPv4            │  IPv6            │     ║
   ║   ├────────┼───────┼──────────────────┼──────────────────┤     ║
   ║   │ opt4   │ 2010  │ 10.11.201.1/24   │ …:2010::1/64     │ OPS ║
   ║   │ opt5   │ 2020  │ 10.11.202.1/24   │ …:2020::1/64     │ DMZ ║
   ║   │ opt6   │ 2030  │ 10.11.203.1/24   │ …:2030::1/64     │ SVC ║
   ║   │ opt7   │ 2040  │ 10.11.204.1/24   │ …:2040::1/64     │ VPN ║
   ║   │ opt8   │ 2100  │ 10.11.210.1/24   │ …:2100::1/64     │ IoT ║
   ║   │ opt9   │ 2110  │ 10.11.211.1/24   │ …:2110::1/64     │ VoIP║
   ║   │ opt10  │ 2200  │ 10.11.220.1/24   │ …:2200::1/64     │ NAS ║
   ║   │ opt11  │ 2300  │ 10.11.230.1/24   │ …:2300::1/64     │ Med ║
   ║   │ opt12  │ 2320  │ 10.11.232.1/24   │ …:2320::1/64     │ GAM ║
   ║   │ opt13  │ 2400  │ 10.11.240.1/24   │ …:2400::1/64     │ CCTV║
   ║   └────────┴───────┴──────────────────┴──────────────────┘     ║
   ║   (IPv6 column abbreviates 2a02:1802:21:NNNN::1/64 from        ║
   ║    Telenet /48; Proximus /56 PD is used for failover only      ║
   ║    via Track-Interface if explicitly enabled per VLAN.)        ║
   ║                                                                ║
   ╚════════════════════════════════════════════════════════════════╝
                          │
                          ▼
                   Internal clients
              (Proxmox SDN VNets, LXCs,
               VMs, NAS, office Wi-Fi)
```

---

## 2. IP allocation — Telenet (WAN2, static)

**Public IPv4** — `213.214.47.216/29` (6 usable hosts)

| IP                  | Role                          | Notes                                              |
|---------------------|-------------------------------|----------------------------------------------------|
| `213.214.47.217`    | Telenet upstream gateway      | Their BNG; static                                  |
| `213.214.47.218`    | pfsense01 (legacy)            | Decommission when replaced by OPNsense MASTER      |
| `213.214.47.219`    | nginx → odoo (legacy)         | Migrate behind Traefik (then free this IP)         |
| `213.214.47.220`    | reserved — HA Phase 2 CARP    | Floating VIP between OPNsense MASTER/BACKUP        |
| `213.214.47.221`    | reserved — HA Phase 2 BACKUP  | OPNsense BACKUP static IP                          |
| `213.214.47.222`    | OPNsense MASTER (current)     | Today's vm-opns-test-01 (will become prod MASTER)  |

**Public IPv6** — `2a02:1802:21::/48` (65,536 /64 subnets)

| Block / Address              | Role                                                              |
|------------------------------|-------------------------------------------------------------------|
| `2a02:1802:21::/64`          | WAN link — Telenet router (`::1`) ↔ our FW                        |
| `2a02:1802:21::1`            | Telenet upstream gateway (always)                                 |
| `2a02:1802:21::2/64`         | **OPNsense WAN2** — Telenet routes the rest of /48 to this address|
| `2a02:1802:21::3/64`         | reserved — HA Phase 2 BACKUP WAN2 IP                              |
| `2a02:1802:21::4/64`         | reserved — HA Phase 2 CARP VIP (alternative to ::2 floating)      |
| `2a02:1802:21:0001::/64`     | LAN (OOB management) sub-prefix                                   |
| `2a02:1802:21:0010::/64`     | OPT1 LAN_TRUNK (informational; no clients)                        |
| `2a02:1802:21:2010::/64`     | VLAN 2010 OPS                                                     |
| `2a02:1802:21:2020::/64`     | VLAN 2020 DMZ                                                     |
| `2a02:1802:21:2030::/64`     | VLAN 2030 SVC                                                     |
| `2a02:1802:21:2040::/64`     | VLAN 2040 VPN                                                     |
| `2a02:1802:21:2100::/64`     | VLAN 2100 IoT                                                     |
| `2a02:1802:21:2110::/64`     | VLAN 2110 VoIP                                                    |
| `2a02:1802:21:2200::/64`     | VLAN 2200 Storage                                                 |
| `2a02:1802:21:2300::/64`     | VLAN 2300 Media                                                   |
| `2a02:1802:21:2320::/64`     | VLAN 2320 GAMING                                                  |
| `2a02:1802:21:2400::/64`     | VLAN 2400 CCTV                                                    |
| `2a02:1802:21:fff0..ffff::/64` | reserved for future use                                         |

**DNS (Telenet provided)**

| Type | Address                      |
|------|------------------------------|
| IPv4 | `195.130.131.11`             |
| IPv4 | `195.130.130.139`            |
| IPv4 | `195.130.130.11`             |
| IPv6 | `2a02:1800:100::1`           |
| IPv6 | `2a02:1800:100::2`           |

---

## 3. IP allocation — Proximus (WAN1, PPPoE dynamic)

**Public IPv4** — dynamic per PPPoE session

| Field                      | Value                                |
|----------------------------|--------------------------------------|
| PPPoE username             | `pu843332@PROXIMUS` (in secret file) |
| PPPoE password             | secret file (rotate via Proximus portal — no API) |
| WAN1 IPv4 (on pppoe0)      | dynamic, rare changes (weeks-months) |
| Tracking record (DDNS)     | `wan1.by-research.be A`              |

**Public IPv6** — `/56` Prefix Delegation via DHCPv6 over PPP

| Block / Address                        | Role                                                |
|----------------------------------------|-----------------------------------------------------|
| `fe80::be24:11ff:fe9f:30e4` (link-local) | OPNsense pppoe0 — only link-local on WAN side     |
| `fe80::22e0:9cff:fe60:4c01` (link-local) | Proximus BRAS — default IPv6 GW                   |
| **`2a02:a03f:6098:9000::/56`** (example PD) | Delegated prefix — used for internal /64 carving via Track-Interface OR NPTv6 (failover only) |
| Inside the /56 (256 /64s available)    | Carved per-VLAN if Telenet down + WAN failover     |

Notes:
- The PD prefix is **dynamic** — Proximus may change it on PPPoE re-dial (rare but possible)
- In normal operation, internal IPv6 uses **Telenet /48** (stable, ours, no renumber risk)
- Proximus /56 PD is **failover-only**: NPTv6 rewrites internal ULA → Proximus prefix on egress when Telenet down
- Tracking record (DDNS): `wan1.by-research.be AAAA` follows current PD assignment

---

## 4. Gateway summary

| Gateway name      | Slot  | Protocol | Address                                | Default | Tier  |
|-------------------|-------|----------|----------------------------------------|---------|-------|
| `WAN2GW`          | opt3  | inet     | `213.214.47.217`                       | YES     | 1     |
| `WAN2GWv6`        | opt3  | inet6    | `2a02:1802:21::1`                      | YES     | 1     |
| `WAN1_PPPoE`      | wan   | inet     | dynamic (Proximus BNG)                 | no      | 2     |
| `WAN1_DHCPv6`     | wan   | inet6    | dynamic (fe80::22e0:9cff:fe60:4c01)    | no      | 2     |

**Gateway groups for failover / policy routing:**

| Group           | Tier 1 (primary)      | Tier 2 (fallback)     | Used by                          |
|-----------------|-----------------------|-----------------------|----------------------------------|
| `GW_DEFAULT_V4` | WAN2GW (Telenet)      | WAN1_PPPoE (Proximus) | Default v4 route (most VLANs)    |
| `GW_DEFAULT_V6` | WAN2GWv6 (Telenet)    | WAN1_DHCPv6 (Proximus)| Default v6 route (most VLANs)    |
| `GW_GAMING`     | WAN1_PPPoE (Proximus) | WAN2GW (Telenet)      | VLAN 2320 GAMING (low-latency)   |
| `GW_VPN_US`     | VPNEXPRESS_US (WG)    | -                     | VLAN 2300 Media (Netflix geo-US) |
| `GW_VPN_FR`     | VPNEXPRESS_FR (WG)    | -                     | (per-host opt-in)                |

---

## 5. Why this design

- **HPE switch handles VLAN tagging upstream of Proxmox** — keeps Proxmox bridges simple (untagged inside), no VLAN-aware bridges needed on most. Proxmox kernel sub-interfaces (`nicN.tag`) terminate the tag.
- **OPNsense PPPoE on raw vNIC (vtnet2)** — Proxmox strips VLAN 10, OPNsense sees plain Ethernet → no double-tagging.
- **OPNsense WAN2 = static** — IPv4 /29 and IPv6 /48 directly on vtnet3, no VLAN sub-interface inside OPNsense (Proxmox already did it).
- **OOB management on bond0 + vmbrOOB** — admin reaches every FW via the OOB subnet (10.6.224.0/20) regardless of WAN state; OPNsense LAN slot gives anti-lockout protection.
- **SDN trunk (vmbrAPPS) = software-only inside Proxmox** — VMs/LXCs attach with their per-VLAN sub-interface; the trunk parent (OPT1) terminates inside OPNsense for routing/firewall.
- **Fabric overlay (vmbrFAB, VLAN 600)** — out-of-band monitoring/orchestration plane that bypasses FW for telemetry collection (Prometheus, Grafana, Ansible heartbeat). Future detail in separate doc.

---

## 6. HA Phase 2 (sketch — not deployed)

```
            ┌──────────────────────────────────┐
            │       HPE V1910-48G switch       │
            └────┬────────────────┬────────────┘
                 │                │
        ┌────────┴────┐    ┌──────┴───────┐
        │ Proxmox A   │    │ Proxmox B    │ ← future second node
        │ srv-proxmox-│    │ srv-proxmox- │
        │   poc-01    │    │   poc-02     │
        └──────┬──────┘    └──────┬───────┘
               │ (each runs one OPNsense VM)
               ▼                  ▼
        ┌──────────────┐   ┌──────────────┐
        │ OPNsense     │   │ OPNsense     │
        │ MASTER (.222 │ ⇄ │ BACKUP (.221 │
        │ + ::3)       │   │ + ::4)       │
        │              │   │              │
        │              │═══│              │ ← pfsync (state replication)
        │              │   │              │
        └──┬───────────┘   └────────────┬─┘
           │                            │
           └──────────┬─────────────────┘
                      │  CARP VIPs on every internal interface:
                      │   - WAN2 .220 + ::2  (clients use this as default GW)
                      │   - LAN VLANs .254 each
                      ▼
                 (clients see only the VIP — failover invisible)
```

Detail in future ADR `infra/00XX-fw-ha-architecture.md`.

---

**Source files:**
- Topology realised in `infra-terraform-proxmox/modules/vm-opnsense/seed/seeds/vm-opns-test-01.json`
- Bridge config in `/etc/network/interfaces` on each Proxmox node (managed by Ansible role `roles/proxmox_network` — pending)
- HPE switch VLAN config — separate doc (TBD in `infra-network-switches/`)
