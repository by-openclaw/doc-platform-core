# ADR-0028 — Firewall Alias and Rule Naming Convention

**Date:** 2026-04-04
**Status:** Accepted
**Deciders:** Youssef Boujraf
**Tags:** `opnsense`, `firewall`, `aliases`, `naming`, `hardening`

---

## Context

OPNsense firewall aliases and rules must be readable, auditable, and maintainable
at scale. Without a naming standard, aliases accumulate with inconsistent names,
rules become ambiguous, and audit reviews require expanding every alias manually
to understand what is allowed.

---

## Decision

### 1. Alias Naming

#### Atomic aliases (preferred — always create these first)

| Type | Pattern | Example | Notes |
|---|---|---|---|
| Network / subnet | `alias_net_{zone}` | `alias_net_mgmt`, `alias_net_vpn` | One per VLAN/zone |
| Single host | `alias_host_{service}` | `alias_host_proxmox`, `alias_host_pihole` | One per significant host |
| Single port | `alias_port_{service}` | `alias_port_ssh`, `alias_port_https` | One per service port |
| URL / GeoIP feed | `alias_url_{feed}` | `alias_url_spamhaus`, `alias_url_geo_be` | Threat intel feeds |

#### Group aliases (use only when a semantic grouping exists)

| Type | Pattern | Example | Semantic test |
|---|---|---|---|
| Network group | `alias_grp_net_{scope}` | `alias_grp_net_internal` | "all internal zones" is a valid policy concept |
| Port group | `alias_grp_port_{stack}` | `alias_grp_port_web` | HTTP+HTTPS are always treated as one service |

**Rule:** A group alias requires a semantic justification — one sentence that names the shared
policy concept. If you cannot state it clearly, use atomic aliases in the rule instead.
Never create group aliases for convenience alone.

#### Prohibited patterns
- No hardcoded IPs or ports in rule fields — always use an alias
- No unnamed or auto-generated aliases
- No aliases named after people or devices (use role/function names)

### 2. Rule Description Convention

Format: `{ACTION} {source}→{destination} {service}`

| Component | Values | Example |
|---|---|---|
| ACTION | `PASS`, `BLOCK`, `REJECT` | `PASS` |
| source | alias name or `any` | `VPN`, `MGMT`, `WAN` |
| destination | alias name, `FW` (firewall itself), or `any` | `MGMT`, `FW` |
| service | service name or protocol | `WireGuard`, `SSH`, `default` |

Examples:
- `PASS WAN→FW WireGuard`
- `PASS VPN→MGMT SSH`
- `PASS VPN→FW UI`
- `PASS MGMT→FW SSH`
- `BLOCK any→any default`
- `PASS MGMT→WAN outbound NAT`

Labels: Rules that are temporary (bootstrap, testing) must include `BOOTSTRAP-TEMP`
in the description. Ansible removes all `BOOTSTRAP-TEMP` rules during hardening phase.

### 3. Per-User vs Zone Rules

**WireGuard peers (and similar multi-user scenarios):**
- Each peer is assigned a tunnel IP from the VPN subnet (`10.100.0.0/24`)
- Firewall rule source = `alias_net_vpn` — covers all peers automatically
- Adding a new peer = add a WireGuard peer entry only. Zero firewall changes.
- Per-peer host alias + dedicated rule **only** when that peer needs an ACL
  that differs from the group (role-based access). This is the exception, not the rule.

### 4. Versioning

Aliases are version-controlled in `tools/opnsense/config/aliases.conf`.
Any alias change must be accompanied by an Ansible role update.
No alias is created manually in the UI without a corresponding Ansible task.

---

## Consequences

- Every rule is human-readable without expanding aliases during audit
- Group aliases exist only when semantically justified — never as shortcuts
- Adding users (WireGuard peers, VMs) does not require firewall rule changes
- BOOTSTRAP-TEMP label makes hardening phase deterministic — Ansible removes exactly those rules
- Full alias inventory is in version control — drift is detectable

---

## References

- ADR-0015: Network/VLAN architecture + WireGuard
- ADR-0027: OPNsense provisioning contract
- `platform-setup/tools/opnsense/docs/fact-matrix.md` — alias table (live state)
- `ansible-platform/roles/opnsense/tasks/aliases.yml` — Ansible implementation
- `ansible-platform/roles/opnsense/tasks/rules.yml` — Ansible implementation
