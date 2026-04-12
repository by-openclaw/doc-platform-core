# ADR-0030 — Automation Naming Convention

**Date:** 2026-04-04
**Status:** Accepted
**Deciders:** Youssef Boujraf
**Tags:** `naming`, `python`, `ansible`, `automation`, `convention`

---

## Context

The BY-SYSTEMS platform uses Python libraries, Ansible collections, Terraform modules, and playbooks to automate infrastructure. Names for repositories, packages, classes, modules, variables, tests, and API-created objects are currently ad-hoc. Without a binding convention, agents and engineers waste time guessing file locations, import paths, and variable names. A single naming formula makes every artifact discoverable mechanically.

---

## Decision

### 1. Repository Naming

Pattern: `{type}-{product}` where `{type}` identifies the artifact kind.

| Type prefix | Artifact | Example |
|---|---|---|
| `lib` | Python library | `lib-opnsense`, `lib-synology-dsm` |
| `ansible` | Ansible collection | `ansible-opnsense`, `ansible-platform` |
| `infra` | Terraform root/module | `infra-terraform-proxmox` |
| `doc` | Documentation | `doc-platform-core` |
| `platform` | Config / runbooks | `platform-setup` |

All lowercase, hyphen-separated. No underscores in repo names.

### 2. Python Package Naming

The installable package name uses hyphens; the importable package name uses underscores (PEP 8).

| Context | Pattern | Example |
|---|---|---|
| `pip install` | `{product}` (hyphens) | `pip install opnsense`, `pip install synology-dsm` |
| `import` | `{product}` (underscores) | `import opnsense`, `import synology_dsm` |
| Local dev | `pip install -e .` | Editable install from repo root |

No `lib-` or `by-` prefix in the package name. The package name is the product name.

### 3. Python Class Naming

| Kind | Pattern | Examples |
|---|---|---|
| Client | `{Product}Client` | `OpnsenseClient`, `DSMClient` |
| Manager | `{Domain}{Entity}Manager` | `AuthUserManager`, `FwAliasManager`, `WgServerManager` |
| Model | `{Entity}` | `AuthUser`, `FwAlias`, `WgServer` (frozen dataclass) |
| Exception | `{Product}{Type}Error` | `OpnsenseAuthError`, `OpnsenseValidationError` |
| Result | `EnsureResult` | Shared across all managers (frozen dataclass) |
| Credential | `{Source}CredentialProvider` | `EnvCredentialProvider`, `VaultCredentialProvider` |

### 4. Python Module (File) Naming

Files use `snake_case`, one file per class or closely-related group.

```
{package}/
  client.py
  exceptions.py
  credentials.py
  models/
    auth_user.py
    fw_alias.py
  managers/
    base.py
    auth_user.py
    fw_alias.py
```

### 5. Ansible Module Naming

Pattern: `{product}_{domain}_{entity}`

Examples:

- `opnsense_auth_user`, `opnsense_auth_group`, `opnsense_auth_priv`
- `opnsense_fw_alias`, `opnsense_fw_rule`, `opnsense_fw_snat`
- `opnsense_ub_forward`, `opnsense_ub_host_override`
- `opnsense_wg_server`, `opnsense_wg_client`
- `opnsense_kea_subnet`, `opnsense_kea_reservation`

Domain abbreviations (consistent across the library and collection):

| Domain | Abbreviation | Covers |
|---|---|---|
| auth | `auth` | users, groups, privileges |
| firewall | `fw` | aliases, rules, NAT, categories, groups |
| unbound | `ub` | DNS forwarders, overrides, ACLs, DNSBL |
| kea | `kea` | DHCP subnets, reservations, peers |
| wireguard | `wg` | servers, clients/peers |
| openvpn | `ovpn` | instances, CSOs |
| ipsec | `ipsec` | connections, children, keys, PSKs |
| interfaces | `if` | VLANs, VIPs, bridges, LAGG, GRE, GIF |
| routes | `rt` | routes, gateways |
| ids | `ids` | policies, rules, rulesets |
| trafficshaper | `ts` | pipes, queues, rules |
| syslog | `syslog` | destinations |
| cron | `cron` | jobs |
| monit | `monit` | alerts, services, tests |
| trust | `trust` | CAs, certs, CRLs |
| core | `core` | tunables, backups, firmware, services |
| diagnostics | `diag` | system info, interface stats, ARP, routes |

### 6. Ansible Variable Naming

Pattern: `{short}_{domain}_{entities}` (plural for lists)

| Variable | Meaning |
|---|---|
| `opn_auth_users` | List of OPNsense users |
| `opn_auth_groups` | List of OPNsense groups |
| `opn_auth_privs` | List of OPNsense privileges |
| `opn_fw_aliases` | List of firewall aliases |
| `opn_fw_rules` | List of firewall rules |
| `opn_wg_servers` | List of WireGuard servers |
| `opn_wg_clients` | List of WireGuard clients |

The `opn_` prefix is the product short name. Each product defines its own short prefix once.

### 7. Test and Playbook Naming

**Test files:**

| Kind | Pattern | Example |
|---|---|---|
| Unit test | `test_{domain}_{entity}.py` | `tests/unit/test_auth_user.py` |
| Integration test | `test_live_{domain}_{entity}.py` | `tests/integration/test_live_auth_user.py` |

**Playbooks:**

| Pattern | Example |
|---|---|
| `playbook_{scope}.yml` | `playbooks/playbook_auth.yml` |
| `playbook_{scope}.yml` | `playbooks/playbook_firewall.yml` |
| `playbook_{scope}.yml` | `playbooks/playbook_dns.yml` |

One playbook per domain scope.

### 8. API Object Naming

Objects created via automation through product APIs follow existing ADRs:

| Object type | Convention source |
|---|---|
| Firewall aliases and rules | ADR-0028 (alias/rule naming patterns) |
| Identity (users, groups, service accounts) | ADR-0010 (naming and identity convention) |
| All other objects | `description` field follows `{ACTION} {context}` pattern |

---

## Consequences

**Positive:**

- All new repos follow these conventions from day 1 -- no naming debates
- Agents can locate any module, class, test, or variable by applying the naming formula mechanically
- Domain abbreviation table is a single source of truth shared between Python libs and Ansible collections
- Consistent variable naming makes Ansible inventory files predictable and reviewable

**Negative:**

- Existing repos (`ansible-platform`, `lib-synology-dsm`) require alignment on next refactor
- Short domain abbreviations (`ub`, `ts`, `rt`) are less discoverable for newcomers -- the abbreviation table must be consulted
- Strict naming limits creative freedom -- every name must fit the pattern or the pattern must be amended via ADR update

---

## References

- ADR-0010: Naming and Identity Convention
- ADR-0028: Firewall Alias and Rule Naming Convention
- ADR-0029: Python Library Design Standard
- PEP 8: Python naming conventions
