# naming/0003 — Firewall Aliases & Rule Descriptions

**Status:** Draft
**Date:** 2026-04-13 (supersedes flat ADR-0028, 2026-04-04)
**Scope:** OPNsense firewall alias names and rule description format.
**Related:** `naming/0001-infra`, `services/0001-opnsense-provisioning` (future, from flat 0027), `infra/0004-network-architecture` (future, from flat 0015+0032)

---

## Context

OPNsense aliases accumulate inconsistent names when created ad-hoc. Rules become ambiguous: a human auditor has to expand every alias to understand what is allowed, and two aliases may overlap silently. This ADR defines a binding pattern for alias names, the rule description format, and the rules around grouping.

## Decision

### 1. Atomic aliases — always create these first

Atomic aliases represent a single concept: one network, one host, one port, one feed. They are the preferred building block.

The **`alias_` prefix is not used** — every object in this section is already typed as an alias by OPNsense at creation (UI, API, or Ansible). The name expresses what the alias *is*, not that it is an alias.

| Kind | Pattern | Example |
|---|---|---|
| Network / subnet | `net_{zone}` | `net_mgmt`, `net_vpn`, `net_dmz` |
| Single host | `host_{service}` | `host_proxmox`, `host_netbox` |
| Single port | `port_{service}` | `port_ssh`, `port_https`, `port_wg` |
| URL / GeoIP feed | `url_{feed}` | `url_spamhaus`, `url_geo_be` |

**Rules:**
- All lowercase, underscore-separated. OPNsense alias names do not accept hyphens reliably across versions — underscore is the safe choice.
- `{zone}` matches the VLAN naming convention (see `infra/0004-network-architecture` when refactored)
- `{service}` is the short service identifier — prefer readable names (`ssh`, `https`) over ports (`22`, `443`) so alias intent is clear without expansion
- One atomic alias per distinct concept. Duplicates (`net_mgmt` + `net_management`) are a violation.
- The first token (`net`, `host`, `port`, `url`) is the **kind**, and it determines the OPNsense alias type (Network, Host, Port, URL). Ansible-managed aliases derive the OPNsense type from this token.

### 1a. Dual-stack family suffix (IPv4 / IPv6)

OPNsense evaluates firewall rules **per address family** — a rule is `inet` or
`inet6`, and an alias referenced by an IPv4 rule must hold IPv4 content. A
dual-stack deployment therefore maintains **separate IPv4 and IPv6 alias
families**. To distinguish them, the kind token MAY carry a `4` / `6` suffix:

| Pattern | Example |
|---|---|
| `net{4,6}_{zone}` | `net4_mgmt`, `net6_mgmt` |
| `grp_net{4,6}_{scope}` | `grp_net4_internal`, `grp_net6_internal` |
| `host{4,6}_{service}` | `host4_fw_gateways`, `host6_fw_gateways` |

**Rules:**
- The suffix is used **only** for the v4/v6 split. A single-family or
  mixed-content alias keeps the unsuffixed form (`host_adguard`, `port_https`,
  `host_public_resolvers`).
- Both families of a zone share the same `{zone}` token so they read as a pair
  (`net4_mgmt` ↔ `net6_mgmt`).
- All other §1 rules still apply (lowercase, one concept per alias, no `alias_`
  prefix, the kind token determines the OPNsense alias type).
- **Rules are dual-stack by intent**: every rule on a dual-stack interface exists
  as an `inet` + `inet6` pair (same description, the v6 twin suffixed `v6`).
- **Single-family interfaces (transitional).** During the pfSense → OPNsense
  migration some interfaces carry one address family only — the OOB management
  network, and the Proximus WAN while Telenet still terminates on the PROD
  pfSense. Rules on those interfaces are legitimately single-family and need no
  twin. The catalog names them explicitly (`opn_single_family_interfaces`); an
  entry is removed the day its interface becomes dual-stack, and the catalog
  lint then demands the twin. This exception disappears with the migration.

### 2. Group aliases — only with semantic justification

Group aliases exist **only** when a shared policy concept can be stated in one sentence.

| Kind | Pattern | Example | Valid when |
|---|---|---|---|
| Network group | `grp_net_{scope}` | `grp_net_internal` | "all internal zones treated as one trust boundary" is a stable policy concept |
| Port group | `grp_port_{stack}` | `grp_port_web` | HTTP + HTTPS are always treated as one service in this deployment |

**Rules:**
- A group alias requires a **one-sentence semantic justification** stored in the alias description field. If the justification cannot be written clearly, use atomic aliases directly in the rule.
- Group aliases exist for **policy clarity**, not for typing convenience. "I don't want to list four aliases" is not a valid justification.
- When a group's membership starts diverging (e.g. an "internal" zone needs to be excluded for one rule), collapse the group and use atomic aliases per rule.

### 3. Prohibited patterns

- **No hardcoded IPs or ports in rule source/destination/port fields.** Always use an alias, even for a single host.
- **No unnamed or auto-generated alias names.** Every alias is created intentionally and named per §1 or §2.
- **No `alias_` prefix in the name.** OPNsense already marks the object as an alias; the prefix adds noise, not information.
- **No aliases named after people or physical devices.** Use role or function names (`host_netbox`, not `host_youssef` or `host_rack3_server2`).
- **No abbreviations that need a lookup table.** `net_mgmt` is fine (`mgmt` is universal); `net_xr4` is not.

### 4. Rule description format

Every rule description follows:

```
{ACTION} {source}→{destination} {service}
```

| Component | Allowed values | Example |
|---|---|---|
| `ACTION` | `PASS`, `BLOCK`, `REJECT` | `PASS` |
| `source` | Alias name or `any` | `net_vpn`, `any` |
| `destination` | Alias name, `FW` (the firewall itself), or `any` | `net_mgmt`, `FW` |
| `service` | Service name or protocol (alias preferred) | `SSH`, `HTTPS`, `WireGuard`, `default` |

**Rules:**
- Rule description is **human-readable without expanding aliases**. An auditor should understand what a rule does by reading the description alone.
- Use `→` (Unicode arrow) or `->` (ASCII fallback). Pick one per deployment and be consistent.
- Use short alias tokens in descriptions, not full alias names — e.g. `PASS VPN→MGMT SSH`, not `PASS net_vpn→net_mgmt port_ssh`. The full alias name is in the rule's Source/Destination field.

**Examples:**

| Description | Meaning |
|---|---|
| `PASS WAN→FW WireGuard` | Accept WireGuard from the internet to the firewall itself |
| `PASS VPN→MGMT SSH` | VPN clients can SSH to management hosts |
| `PASS VPN→FW UI` | VPN clients can reach the OPNsense web UI |
| `PASS MGMT→FW SSH` | Management network can SSH to the firewall |
| `BLOCK any→any default` | Default deny |
| `PASS MGMT→WAN outbound NAT` | Management hosts can reach the internet via NAT |

### 5. Temporary / bootstrap rules — `BOOTSTRAP-TEMP` label

Rules added during platform bootstrap or one-off testing that must be removed later include `BOOTSTRAP-TEMP` in the description:

```
PASS any→FW BOOTSTRAP-TEMP SSH (initial provisioning)
```

**Rules:**
- The hardening Ansible role removes **all** rules whose description contains `BOOTSTRAP-TEMP` during the hardening phase. This makes the hardening phase deterministic.
- Never use `BOOTSTRAP-TEMP` for a rule that should survive hardening.

### 6. Per-user vs zone rules

When multiple users share the same trust profile (e.g. WireGuard peers), **use zone rules, not per-user rules.**

- Each WireGuard peer receives a tunnel IP from the VPN subnet (`net_vpn`)
- The firewall rule's source is `net_vpn` — it covers all peers automatically
- Adding a new peer = add a WireGuard peer entry. **Zero firewall rule changes.**

**Exception:** a per-peer host alias + dedicated rule is allowed **only** when that peer needs ACLs different from the group (role-based access). This is the exception, not the rule. Document the justification in the alias description.

### 7. Version control

Aliases are version-controlled in the Ansible role that manages OPNsense (see `services/0001-opnsense-provisioning`). No alias is created manually in the OPNsense web UI without a corresponding Ansible task. Drift from Ansible state is detected and reconciled per the provisioning ADR.

## Consequences

- Every rule is human-readable without expanding aliases during audit
- Group aliases exist only when semantically justified — never as typing shortcuts
- Adding users (WireGuard peers, new VMs) does not require firewall rule changes
- `BOOTSTRAP-TEMP` label makes hardening phase deterministic
- Alias inventory lives in version control; drift is detectable
- Prohibition on hardcoded IPs means a network renumbering is an alias update, not a rule rewrite

## CISO mapping

| Framework | Controls covered |
|---|---|
| ISO 27001:2022 | A.8.9 (configuration management), A.8.20 (networks security), A.8.32 (change management) |
| NIS2 | Art. 21(2)(e) (network and information systems security) |
