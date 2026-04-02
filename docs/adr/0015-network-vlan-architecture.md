# ADR-0015: Network VLAN Architecture

- **Status:** Draft
- **Date:** 2026-04-02
- **Deciders:** @yboujraf

## Context

The PoC platform spans multiple network segments (OOB, POC-MGMT, POC-SVC, POC-DHCP) defined in ADR-0006. A formal decision record is needed to capture VLAN IDs, Proxmox SDN zone configuration, and OPNsense firewall rules so that the network is reproducible and auditable.

## Decision

TODO: pending @yboujraf review

### OPNsense Firewall Rules Standard (decided 2026-04-02)

**All firewall rules use named aliases. Hardcoded IPs, ports, and URLs are banned in rules.**

Alias types and naming convention:

| Type | Naming pattern | Example | Contains |
|---|---|---|---|
| Host | `alias_host_{service}` | `alias_host_vault` | IP(s) of the VM/service |
| Port | `alias_port_{service}_{proto}` | `alias_port_vault_api` | Port or port range |
| Network | `alias_net_{vlan}` | `alias_net_mgmt` | VLAN subnet CIDR |
| URL/feed | `alias_url_{feed}` | `alias_url_spamhaus` | Threat feed / blocklist URL |

Rule format:
```
pass in on {interface} from {alias_net_src} to {alias_host_dst} port {alias_port_dst}
```

Example — allow management zone to reach Vault API:
```
pass in on VLAN_PLATFORM from alias_net_mgmt to alias_host_vault port alias_port_vault_api
```

Never:
```
pass in on VLAN300 from 10.6.225.0/24 to 10.6.225.10 port 8200
```

Alias definitions live in `tools/opnsense/config/aliases.conf` (exported from OPNsense). All alias names follow the platform naming convention.

## Consequences

TODO: pending full decision
- Rules become self-documenting — alias names communicate intent
- IP changes require updating one alias, not hunting through rules
- Alias file becomes source of truth for firewall topology
- All tool `docs/network.md` files must document rules using alias format

## References

- ADR-0006 §2 — IP addressing plan and VLAN table
- `docs/stack.md` — OPNsense, Proxmox SDN entries
- `docs/2026-04-01-infra-bom-poc-proxmox.md` — PoC deployment snapshot
- `brainstorming/2026-04-02-doc-matrix.md` — missing ADR identified here
</content>