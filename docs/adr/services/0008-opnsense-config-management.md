# services/0008 — OPNsense config management: seed vs Ansible MVC boundary

**Status:** Draft
**Date:** 2026-06-02
**Scope:** Defines which parts of an OPNsense firewall's configuration are produced by the **seed** (rendered `config.xml`) versus applied live by **Ansible over the MVC API**, the rule that draws the line, and the procedure (with blast radius) for changing config — notably adding a VLAN. Does not redefine VLAN/IP assignments themselves (that data lives in `infra/0004` → NetBox) or the Terraform/bootstrap ownership contract (`services/0001`).
**Deciders:** @yboujraf
**Related:** `services/0001-opnsense`, `infra/0004-network-architecture`, `services/0003-netbox-cmdb`, `naming/0003-firewall`, `security/0001-secret-storage`

---

## Context

OPNsense (FreeBSD appliance) exposes an MVC REST API, but on the deployed build (26.1.x) that API **cannot do everything**. Concretely, verified on the live FW:

| Operation | MVC API |
|---|---|
| VLAN **device** create (`Interfaces → Devices → VLAN`) | ✅ `/api/interfaces/vlan_settings/*` |
| Interface **assignment** (`Interfaces → Assignments`, device → `optN`) | ❌ `addInterface` → 404 |
| Interface **IP** config (`Interfaces → [name]`, v4/v6) | ❌ `setInterface` → 404 |
| Gateways | ✅ `/api/routing/settings` |
| Firewall rules, aliases, source-NAT | ✅ |
| Kea DHCP (v4/v6) | ✅ |
| Unbound / dnscrypt | ✅ |
| NetFlow capture enable (`setconfig`) | ❌ rejected on this build |
| CrowdSec | ✅ |

Because interface **assignment + IP** and **NetFlow capture** have no working API, they can only be set in `config.xml` — i.e. the **seed**. Repeated sessions lost the working firewall config because a *minimal* seed (4 interfaces only) was deployed and the assumption "Ansible MVC will add the rest" was false for those operations. This ADR fixes the boundary so it cannot be lost again, and so config is **generated** (reproducible), never restored from a frozen backup whose per-FW details (IPs, certs, keys) would be wrong on the next FW.

## Decision

**Draw the seed/Ansible boundary by API capability, and generate the seed — never restore a backup.**

> **The rule:** if the OPNsense MVC API can create/modify it **idempotently**, it belongs to **Ansible MVC**. If it cannot (interface assignment, interface IP, NetFlow capture), it belongs to the **seed**.

### Layer 1 — Seed (rendered `config.xml`)

The seed is **generated** by `build-seed.py` from a per-FW source (inventory + vault today; **NetBox** in Phase 2 — `services/0003`). It is **not** a saved backup. It carries everything the API cannot:

- physical interfaces (LAN/trunk), **WAN1** (Proximus PPPoE) and **WAN2** (Telenet static — physical NIC, e.g. `vtnet3`)
- **all VLANs**: device + assignment (`optN`) + IPv4/IPv6 — the full 3-step VLAN object
- OOB/MGMT reachability, SSH key + API key/cert (so Ansible can reach the FW), base system

All per-FW values (Telenet `/29` + `/64`, PPPoE creds, admin hash, API keys) are injected at render time from the secret store (`security/0001`) — **never inlined in git or docs**.

### Layer 2 — Ansible MVC (live, idempotent, over the API)

Everything that only **references an already-existing interface**, applied by the `by_systems.opnsense` / `ansibleguy.opnsense` roles from the catalog:

- gateways (incl. WAN failover groups), firewall rules, aliases, source-NAT
- **Kea dual-stack** DHCP, Unbound + dnscrypt, NetFlow aggregation, CrowdSec

This layer is changed with an Ansible run — **no reseed**.

### VLAN lifecycle (the three steps, and where each lives)

| Step | OPNsense GUI | API? | Lives in |
|---|---|---|---|
| 1. VLAN device (`vtnet1.1010`) | Interfaces → Devices → VLAN | ✅ | seed (or Ansible) |
| 2. Assignment (`vtnet1.1010` → `opt2` = MGMT) | Interfaces → Assignments | ❌ | **seed** |
| 3. IP (v4/v6) | Interfaces → [MGMT] | ❌ | **seed** |

A VLAN is therefore **born in the seed** (all three steps); everything it carries (DHCP, rules, routing) is **live Ansible MVC**.

### Changing config — blast radius

- **Add / change a VLAN** (touches Layer 1): regenerate the seed from the source → apply via the **backup/restore (config import) API** → OPNsense runs an *interfaces reconfigure* (a few seconds' reload, established sessions generally survive) → **then re-run Ansible MVC**. This is **not** an OS reinstall, and existing VLANs/services are preserved.
- **Change rules / DHCP / gateways / DNS** (Layer 2 only): a plain idempotent Ansible run. No reload of interfaces, no reseed.
- **Full reseed (OS reinstall)**: only for building a **new** FW, never for a config change.

### Drift rule (mandatory)

After **any** config import, **re-run Ansible MVC**. It is idempotent and re-asserts gateways/rules/Kea/NetFlow/CrowdSec for all VLANs (old and new), so a config import can never silently drop the Layer-2 state.

### Reproducibility gate

A fresh FW must reach full working state **exclusively via seed + Ansible MVC**, proven by `scripts/fw_verify_health.py` (ansible-platform) going **green on a throwaway VM, twice, timed**, before any production window. The gate checks: all interfaces up, both WANs up with correct IPs, gateways up, DNS chain + CrowdSec + Kea + reporting services running.

## Consequences

**Positive**
- The running config can no longer be silently lost — the boundary is explicit and the seed is generated, not minimal.
- Config is reproducible and per-FW-parameterized; a new FW gets its own details with no hardcoding.
- Day-2 changes (rules, DHCP, routing) are live and non-disruptive.

**Negative / accepted**
- Adding a VLAN needs a seed re-render + config import (a brief interfaces reload), because OPNsense has no API to assign an interface. It is not hot-add, but it is not a reinstall either.
- The seed must always represent the **complete** L2/L3; the drift rule (re-run MVC after import) is required to stay safe.

**Open item**
- Confirm on the **test** FW (never prod) whether this build has a granular interface-assignment API. If one exists, Step 2/3 move to Ansible and VLANs become hot-add — this ADR is then revised.

**Phase 2**
- NetBox becomes the IPAM source of truth (`services/0003`); the seed render and the Ansible catalog both read from it, making "add a VLAN in NetBox → regenerate seed + Ansible run" the standard flow.

## CISO mapping

- **ISO/IEC 27001:2022 A.8.9 (Configuration management):** firewall configuration is defined as code (seed + catalog), reproducible and reviewable; no undocumented live-only state.
- **ISO/IEC 27001:2022 A.8.32 (Change management):** Layer-1 vs Layer-2 change procedures with explicit blast radius; verification gate before production.
- **NIS2 (2022/2555) Art.21:** reproducible, verifiable security-control baseline on the network edge.
- **GDPR (2016/679) Art.32:** consistent enforcement of segmentation/filtering controls protecting personal data, re-asserted idempotently after every change.

## References

- `services/0001-opnsense` — Terraform / bootstrap / Ansible ownership contract
- `infra/0004-network-architecture` — VLAN registry, addressing, WAN uplinks
- `services/0003-netbox-cmdb` — Phase 2 IPAM source of truth
- `security/0001-secret-storage` — per-FW secret injection
- ansible-platform `scripts/fw_verify_health.py` — reproducibility gate
- infra-terraform-proxmox `modules/vm-opnsense/seed/build-seed.py` — seed renderer
