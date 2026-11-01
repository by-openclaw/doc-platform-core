# infra/0014 — Routing Domains, VIPs, Firewall Pair, IPAM and SDN

**Status:** Draft (proposal — approve by merging)
**Date:** 2026-11-01
**Scope:** The layer-3 design of the platform network: routing domains (VRF) and what may leak between them, virtual addresses (VIP) and who owns them, the firewall pair's place in the topology, the IP address management of record, and what Proxmox SDN is asked to do now and with the cluster. Builds on `infra/0004` (zones, VLAN registry, SDN zone naming — unchanged, two rows added) and `infra/0012` (placement, firewall pair, order of work). Not in scope: firewall rule content (`naming/0003`, the catalog), the switching hardware purchase (BOM), WireGuard/NetBird peers.
**Related:** `infra/0004-network-architecture`, `infra/0009-hypervisor-cluster`, `infra/0011-resolver-high-availability`, `infra/0012-workload-placement-and-availability-classes`, `services/0003-netbox-cmdb`, `services/0001-opnsense`, `services/0008-opnsense-config-management`

---

## Context

**Measured on 2026-11-01 (one hypervisor, one firewall VM).** The firewall is the only router: it holds `.1` and `::1` in each of the nine production zones and the nine test zones of `infra/0004`. Proxmox SDN has two VLAN zones (`prod`, `test`) on the application bridge over a bond, eighteen vnets, subnets as metadata only, IPAM plugin `pve`. Four more bridges exist outside SDN: two ISP uplinks, the fabric management bridge (VLAN 600, 10 GbE) and the out-of-band bridge. One virtual address exists (the resolver, keepalived VRRP, `infra/0011`). There is no CARP, no routing domain anywhere, no leak, and the fabric management VLAN's gateway and DHCP still live on the legacy firewall. The address plan lives in a YAML catalog in the configuration repository; NetBox holds nothing. The core switch is a 48-port gigabit unit without 10 GbE. One zone (`gaming`) exists in the catalog without a vnet.

**Owner direction, 2026-11-01:** work the network architecture in parallel with the BOM — "VLAN, VRF, VRF leaks, VIP, firewall, IPAM, SDN". The on-premise cloud of `infra/0012` (tenant workloads such as Odoo instances next to the platform) is the driver: tenants need isolation from each other and a controlled path to shared services and to the internet.

## Decision

### 1. Routing domains

A routing domain is a NetBox VRF with a route distinguisher and route targets, whether it is enforced by firewall policy today or by EVPN tomorrow. Five kinds:

| Domain | Holds | Leaks | Router |
|---|---|---|---|
| **platform** | the eleven zones of `infra/0004` (MGMT, DMZ, SVC, VPN, IoT, VoIP, Storage, Media, CCTV and the test mirror) | none between zones except through the firewall's rules | firewall pair |
| **shared** | what every tenant may consume read-only: the resolver address, NTP, package mirrors, the identity front door — VLAN 1050 (`infra/0004` §3, new row) | imported by every tenant domain (the only leak) | firewall pair (gateway), reachable without it once EVPN leaks it |
| **tenant-<name>** | one per customer or workload group — VLANs 1600–1690, one /24 each (`infra/0004` §3, new rows); test mirror 2600–2690 | imports `shared` only; everything else leaves through the firewall (NAT, IDS, rules) | anycast gateway of its EVPN zone (cluster phase), firewall before that |
| **mgmt-oob** | out-of-band and fabric management (`platform_mgmt_zones` of the catalog) | none; entered only through the jump host | the node bridges and the firewall's management leg; never through SDN |
| **transit** | one VLAN per ISP handoff (`infra/0004` §3, new rows), ISP-addressed | none | the firewall pair only (CARP virtual address per ISP) |

Leak policy, in one sentence: a tenant sees `shared` and nothing else; everything else is a firewall decision, logged. The `shared` domain therefore holds only services built for many tenants (host firewall on each, authentication on each, no tenant data), and nothing is added to it without an ADR line.

### 2. Where the domains are enforced

| Phase (`infra/0012` §6) | platform / shared | tenants |
|---|---|---|
| **now** (one node, one firewall) | firewall rules between zones (as today) | a VLAN zone vnet per tenant on the trunk, gateway on the firewall, rules deny tenant↔tenant and tenant↔platform except `shared` |
| **Phase 0–2** (cluster, 10 GbE fabric, firewall pair) | unchanged: VLAN zones, firewall pair routes | Proxmox SDN **EVPN zones**: one VRF per tenant zone, anycast gateway on every node, VXLAN over the fabric, `exit nodes` = the two nodes carrying the firewall pair, route-target import of `shared` only (Proxmox SDN EVPN: VRF per zone, exit nodes, route-target import) |

OPNsense has no routing-domain feature beyond gateway groups, so the firewall never carries a VRF: it is the exit of every domain and the policy point between them. The switch carries the fabric VRF (Arista decision of 2026-09) and the transit VLANs; it routes nothing of the platform.

### 3. Virtual addresses

One mechanism per layer, nothing else:

| Layer | Mechanism | Rule |
|---|---|---|
| Firewall | CARP (OPNsense, with pfsync on VLAN 1900 and configuration sync) | one virtual address per interface the pair serves, incl. each transit VLAN |
| Linux services (resolver, ingress pair, database endpoint pair, …) | keepalived VRRP (`roles/keepalived`: unicast peers, host firewall accepts VRRP from the peer only, no VRRP password — RFC 5798 has none) | the service's clients get the VIP, never a member |
| Kubernetes | kube-vip in layer-2 mode (Phase 2 of `infra/0012`) | one VIP per exposed service |

Every VIP is a NetBox IP address with role `vip`, the owning service and the mechanism as tags, and lives in the block **.240–.254** of its /24 (reserved for VIPs in every zone). The resolver VIP at `.100` (`infra/0011`) is the recorded exception and stays.

### 4. The firewall pair in the topology

As decided in `infra/0012` §4: CARP active/passive on two nodes, one address per node plus the virtual address per interface, pfsync on its own VLAN (1900 `sync`), configuration sync from master to backup, every ISP handoff on a transit VLAN trunked to every node so the pair can run on any two. The pair stays the only router of the platform domain and the exit of every tenant domain; its rules stay the catalog applied through `lib-opnsense` (`services/0008`), and from `services/0003` Phase 4 its aliases are rendered from NetBox. The fabric management VLAN's gateway and DHCP move from the legacy firewall to the pair before the legacy unit is retired.

### 5. IPAM of record

NetBox (`services/0003`) is the IPAM; the YAML catalog becomes a generated file. Loaded in the CMDB Phase 1:

- Aggregates: the production and test internal ranges of `infra/0004`, their IPv6 ULA prefixes, the management ranges, and the container ranges that must never be reused (Docker pools, the Docker bridge, the k3s pod and service ranges, the VPN overlay) as prefixes with role `container` or `overlay`.
- VLAN groups: `reserved` (1–999 with the owners of `infra/0004` §2), `prod` (1001–1999), `test` (2001–2999).
- VRFs: the five kinds above; prefixes attached to their VRF.
- IP addresses: every guest address from the inventory and Terraform; every VIP; the firewall's addresses; DHCP pools as ranges.

Allocation plan of every /24 (loaded as NetBox ranges with roles):

| Block | Role | Note |
|---|---|---|
| .1 | gateway | the firewall's CARP virtual address |
| .2–.3 | gateway members | the two firewall nodes |
| .4–.9 | infra reserved | — |
| .10–.49 | platform static | assigned by the CMDB |
| .50–.99 | DHCP | Kea pool, as today |
| .100–.239 | static guests | today's assignments fit here unchanged |
| .240–.254 | VIPs | §3 |

Proxmox SDN's IPAM plugin changes from `pve` to `netbox` (vendor plugin) once the prefixes are loaded, so a vnet and its subnets are the same object in both. Terraform and Ansible read addresses from NetBox from `services/0003` Phase 3; until then the catalog YAML is the source and the CMDB load mirrors it.

### 6. SDN

- VLAN zones (`prod`, `test`) stay for the platform domain: layer 2 to the firewall, as `infra/0004` §6.
- Tenant domains: VLAN zone vnets now (one VLAN each), EVPN zones with the cluster (FRR controller on every node, VXLAN underlay on the fabric VLAN with MTU 9000 and vnet MTU 8950, exit nodes = the firewall pair's nodes).
- Every zone, vnet and subnet is declared by Terraform (`modules/sdn`) from the IPAM data, never in the UI.
- Found during the measurement: the `gaming` zone exists in the catalog and in the firewall but has no vnet — it gets its vnet or leaves the catalog in the same pull request as this ADR's first apply.

### 7. Switching, for the BOM

The current 48-port gigabit switch has no 10 GbE and no routing-domain feature: it stays the interim access switch. The BOM item for Phase 0 is the 10 GbE switch of the 2026-09 decision (Arista core, VRF-capable, every node port a trunk carrying the platform, tenant, transit, sync and fabric VLANs) plus the campus PoE and access-point tier. Administrative logins through RADIUS against Authentik (built 2026-10-04) apply to every switch.

### 8. Test tier

Everything above exists twice: the test mirror (VLANs 2xxx, `10.11.x` and `fd11:x::` of `infra/0004` §4) is where every change of this ADR is applied first — the EVPN zone, the leak, the VIP block, the CARP pair — before the production domain.

## Consequences

- `infra/0004` §3 gains the rows below (same VLAN↔subnet mnemonic: VLAN 1XYZ ↔ `10.1.XY.0/24`); §2 is unchanged.
- The catalog YAML of zones becomes a generated artefact once NetBox is loaded; until then it carries the new zones.
- `roles/keepalived` consumers move their VIPs into the `.240–.254` block when they are created (the resolver stays at `.100`); the firewall catalog gains the CARP objects and the transit interfaces with the pair.
- Tenant onboarding = one NetBox VRF + VLAN + prefix + the SDN vnet + the firewall rule set, from one playbook (`infra/0012` Phase 5).
- Proxmox SDN EVPN and the `netbox` IPAM plugin are new moving parts on the nodes: both are exercised on the test tier first.

## Alternatives considered

- **Routing domains on the firewall (FreeBSD FIBs).** Rejected: OPNsense exposes gateway groups, not VRFs; policy on the firewall is the enforcement, the domains live in NetBox and in SDN.
- **Layer 3 on the access switch.** Rejected: the current switch cannot, and the platform's policy point must stay the firewall.
- **One flat VLAN per tenant with firewall rules only, forever.** Kept for "now"; rejected as the end state because every tenant↔shared flow would cross the firewall and every node would need every tenant VLAN on the trunk.
- **Kubernetes CNI (Cilium) as the tenant network.** Deferred to pods (`infra/0012` Phase 3); VMs and containers of tenants need a hypervisor-level domain.

## CISO mapping

| Framework | Controls covered |
|---|---|
| ISO 27001:2022 | A.8.20 (networks security — domains, policy point), A.8.21 (security of network services — VIPs with owners, pair), A.8.22 (segregation of networks — tenant domains, leak policy), A.8.9 (configuration management — IPAM of record, SDN from code), A.8.12 (data leakage prevention — no tenant↔tenant path) |
| NIS2 | Art. 21(2)(e) network security, Art. 21(2)(i) access control policies |

## Revision triggers

- The first tenant is onboarded (the EVPN design meets reality).
- The 10 GbE switch is chosen (fabric VLAN and MTU values fixed).
- NetBox becomes the source for Terraform (`services/0003` Phase 3): the catalog YAML is retired.
- Proxmox changes its SDN EVPN or IPAM plugin model.
