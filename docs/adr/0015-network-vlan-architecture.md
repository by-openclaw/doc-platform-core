# ADR-0015: Network VLAN Architecture

- **Status:** Accepted
- **Date:** 2026-04-02
- **Deciders:** @yboujraf

## Context

The PoC platform spans multiple network segments managed via Arista 7060/7020 switches and OPNsense as the virtual router. A formal VLAN registry is required to ensure the topology is reproducible, auditable, and aligned with Proxmox SDN zone configuration. Earlier planning introduced VLAN IDs 340 and 350 that were subsequently removed; this ADR captures the authoritative active set.

## Decision

### Active VLANs (PoC)

| VLAN ID | Name | Subnet | Purpose |
|---|---|---|---|
| 300 | OOB | — | Out-of-band management (physical infra access) |
| 310 | MGMT | 10.1.1.0/24 | Platform management — Pi-hole, Unifi, jumphost |
| 320 | DMZ | 10.1.2.0/24 | Public-facing — Traefik ingress |
| 330 | SVC | 10.1.3.0/24 | Internal platform services |
| 400 | PROD-MGMT | — | Production management (reserved, not yet active) |
| 410 | PROD-SVC | — | Production services (reserved, not yet active) |

**Supernet:** `10.1.0.0/20` covers all PoC segments.

### Removed VLANs

**VLAN 340 and 350 are RESERVED/NOT USED.** They have been removed from the Arista trunk allowed list. Do not re-allocate these IDs without a new ADR.

### Storage and Backup Networks

No dedicated VM-level storage or backup VLAN. Storage traffic (NAS, Proxmox backup) uses the OOB/MGMT network. A dedicated storage VLAN is a Phase 2 consideration only.

### WAN Uplinks

| Uplink | Status | Role |
|---|---|---|
| OOB prod primary | Active | Primary WAN |
| Proximus | Active | Secondary WAN (failover) |
| Telenet | Untested | Not in active rotation |

### OPNsense Firewall Rule Standard

All firewall rules use named aliases. Hardcoded IPs, ports, and URLs are banned in rules.

| Alias type | Naming pattern | Example |
|---|---|---|
| Host | `alias_host_{service}` | `alias_host_vault` |
| Port | `alias_port_{service}_{proto}` | `alias_port_vault_api` |
| Network | `alias_net_{vlan}` | `alias_net_mgmt` |
| URL/feed | `alias_url_{feed}` | `alias_url_spamhaus` |

Alias definitions are exported to `tools/opnsense/config/aliases.conf` and version-controlled.

## Consequences

- VLAN 340 and 350 must not appear in switch configs, Proxmox SDN zones, or OPNsense interfaces.
- All VM provisioning uses the `10.1.x.x` runtime addressing from the BoM. The `10.6.225.x` range is a temporary bootstrap-only range (pre-SDN) and must not be used as canonical service addressing.
- Firewall rule reviews must verify alias coverage before any new service is deployed.
- OPNsense alias file is the source of truth for firewall topology.

## References

- ADR-0006 §2 — IP addressing plan and layer model
- `docs/stack.md` — OPNsense, Proxmox SDN entries
- `brainstorming/2026-04-02-opus-batch1-feedback.md` — bootstrap vs runtime IP clarification
