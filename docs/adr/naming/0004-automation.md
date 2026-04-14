# naming/0004 — Automation Naming

**Status:** Draft
**Date:** 2026-04-13 (supersedes flat ADR-0030, 2026-04-04)
**Scope:** Patterns for repositories, Python packages/classes/modules, Ansible modules/variables, Terraform resources, tests, and playbooks.
**Related:** `naming/0001-infra`, `naming/0003-firewall`, `lib/python/0001-design-standard` (future, from flat 0029), `git/0002-platform-strategy`

---

## Context

Python libraries, Ansible collections, Terraform modules, and playbooks are the automation surface of the platform. Without a binding pattern, agents and engineers waste time guessing file locations, import paths, and variable names. One naming formula makes every artifact discoverable mechanically.

## Decision

### 1. Repository naming

Pattern: `{type}-{product}` — all lowercase, hyphen-separated, no underscores.

| Type | Artifact | Example |
|---|---|---|
| `lib` | Python library | `lib-opnsense`, `lib-synology-dsm` |
| `ansible` | Ansible collection | `ansible-opnsense`, `ansible-platform` |
| `infra` | Terraform root or module repo | `infra-terraform-proxmox`, `infra-terraform-contabo` |
| `doc` | Documentation | `doc-platform-core` |
| `platform` | Platform config / runbooks (monorepo) | `platform-setup` |
| `svc` | Application service — own code, CI/CD, releases | `svc-orders-api`, `svc-media-gateway` |
| `k8s` | Kubernetes manifests + Helm charts | `k8s-platform`, `k8s-svc-orders-api` |
| `ci-templates` | Shared GitLab CI pipeline templates | `ci-templates` (single repo, no suffix) |

**Rules:**
- One repo per (type, product) pair
- No `by-` or org-prefix in repo name (org is set by GitHub/GitLab path, not by naming)
- Hyphens only — no underscores, no CamelCase
- **`platform-setup` is a monorepo** — all tool configs, runbooks, install scripts, security hardening, and log configs live under `platform-setup/tools/{tool-name}/`. Each tool follows the same internal folder structure (`config/`, `scripts/`, `security/`, `runbooks/`, `log/`). Docker Compose files for that tool are colocated. Adding a new tool = new folder under `platform-setup/tools/`, not a new repo.
- **`svc-*` repos are per-service**, not monorepo. Each application service owns its own code, its own CI/CD pipeline, its own releases, and its own `docs/` (architecture, runbook, API). Service documentation stays in the service repo — `doc-platform-core` covers platform-level docs only.
- **`k8s-*` repos** hold Kubernetes manifests and Helm charts. One per application or per service cluster — never a catch-all "`k8s` monorepo".
- **`ci-templates`** is a single repo at the org level containing shared GitLab CI pipeline templates. Every `svc-*` and `infra-*` repo includes from this repo via the `include:` directive (see `git/0002-platform-strategy §CI templates include pattern`).

### 2. Python package naming

The installable package name uses hyphens; the importable package name uses underscores (PEP 8).

| Context | Pattern | Example |
|---|---|---|
| `pip install` | `{product}` (hyphens) | `pip install opnsense`, `pip install synology-dsm` |
| `import` | `{product}` (underscores) | `import opnsense`, `import synology_dsm` |
| Local dev | `pip install -e .` | Editable install from repo root |

No `lib-` or `by-` prefix in the package name. The package name is the product name — `lib-` is a **repository** prefix, not a package prefix.

### 3. Python class naming

| Kind | Pattern | Examples |
|---|---|---|
| Client | `{Product}Client` | `OpnsenseClient`, `DSMClient` |
| Manager | `{Domain}{Entity}Manager` | `AuthUserManager`, `FwAliasManager`, `WgServerManager` |
| Model | `{Entity}` | `AuthUser`, `FwAlias`, `WgServer` (frozen dataclass) |
| Exception | `{Product}{Type}Error` | `OpnsenseAuthError`, `OpnsenseValidationError` |
| Result | `EnsureResult` | Shared across all managers (frozen dataclass) |
| Credential provider | `{Source}CredentialProvider` | `EnvCredentialProvider`, `VaultCredentialProvider` |

Library design details (base classes, method signatures, error hierarchy) are in `lib/python/0001-design-standard` — this ADR covers names only.

### 4. Python module (file) naming — hierarchical, not flat

Files use `snake_case`. The package layout is **hierarchical by domain**, not flat. One folder per domain, one file per entity. Import paths are dotted and reflect the folder structure.

```
src/{package}/
├── client.py
├── exceptions.py
├── credentials.py
├── validators.py
├── logging.py
├── core/                     ← shared internals (base manager, diff engine, ...)
├── models/
│   ├── auth/
│   │   └── user.py           ← class AuthUser
│   ├── firewall/
│   │   └── alias.py          ← class FwAlias
│   └── ...
└── managers/
    ├── base.py               ← class BaseManager (shared)
    ├── auth/
    │   ├── user.py           ← class AuthUserManager
    │   ├── group.py          ← class AuthGroupManager
    │   ├── priv.py
    │   └── api_key.py
    ├── firewall/
    │   ├── alias.py          ← class FwAliasManager
    │   └── rule.py
    └── vpn/
        ├── wireguard/
        └── openvpn/
```

Import example:
```python
from opnsense.managers.auth.user import AuthUserManager
from opnsense.models.firewall.alias import FwAlias
```

**Rules:**
- One file per class (or closely related group of dataclasses/helpers)
- Folder = domain (see §6 abbreviation table); file = entity
- File names are `snake_case` (PEP 8); class names are `CamelCase` (§3)
- Reference implementation: [`lib-opnsense/src/opnsense/managers/`](https://github.com/by-openclaw/lib-opnsense)

### 5. Ansible module naming — flat snake_case (Ansible convention, mandatory)

Pattern: `{product}_{domain}_{entity}`

Examples:
- `opnsense_auth_user`, `opnsense_auth_group`, `opnsense_auth_priv`, `opnsense_auth_api_key`
- `opnsense_fw_alias`, `opnsense_fw_filter`, `opnsense_fw_dnat`, `opnsense_fw_source_nat`
- `opnsense_ub_forward`, `opnsense_ub_host_override`
- `opnsense_wg_server`, `opnsense_wg_client`
- `opnsense_kea_subnet`, `opnsense_kea_reservation`

**Ansible collections require flat module files** in `plugins/modules/`. Nested module paths are not supported in stable collections. The flat `{product}_{domain}_{entity}` name encodes what the nested folder structure encodes in the Python library.

**Two ecosystems, two patterns — both correct:**

| Layer | Form | Example |
|---|---|---|
| Python library (hierarchical) | Dotted, folder-per-domain | `opnsense.managers.auth.user.AuthUserManager` |
| Ansible collection (flat, convention-forced) | Underscored, one file per entity in `plugins/modules/` | `plugins/modules/opnsense_auth_user.py` |

One-to-one mapping: the Ansible module `opnsense_auth_user` is a thin wrapper around `opnsense.managers.auth.user.AuthUserManager.ensure()`. Every Ansible module corresponds to exactly one Python manager.

### 6. Domain abbreviation table

Shared across Python libraries and Ansible collections. Adding a new domain requires an ADR amendment — consistency across the platform is worth more than one-off creativity.

| Domain | Abbreviation | Covers |
|---|---|---|
| auth | `auth` | Users, groups, privileges |
| firewall | `fw` | Aliases, rules, NAT, categories, groups |
| unbound | `ub` | DNS forwarders, overrides, ACLs, DNSBL |
| kea | `kea` | DHCP subnets, reservations, peers |
| wireguard | `wg` | Servers, clients / peers |
| openvpn | `ovpn` | Instances, CSOs |
| ipsec | `ipsec` | Connections, children, keys, PSKs |
| interfaces | `if` | VLANs, VIPs, bridges, LAGG, GRE, GIF |
| routes | `rt` | Routes, gateways |
| ids / ips | `ids` | Policies, rules, rulesets |
| traffic shaper | `ts` | Pipes, queues, rules |
| syslog | `syslog` | Destinations |
| cron | `cron` | Jobs |
| monit | `monit` | Alerts, services, tests |
| trust | `trust` | CAs, certificates, CRLs |
| core | `core` | Tunables, backups, firmware, services |
| diagnostics | `diag` | System info, interface stats, ARP, routes |

### 7. Ansible variable naming — flat snake_case (Ansible/YAML, no dots)

Pattern: `{short}_{scope}_{name}` — flat snake_case, lowercase.

Ansible variables are flat snake_case tokens. Dots are not used because YAML/Jinja treats `foo.bar` as attribute access, not a variable name. Nested/dotted variable names break `{{ ... }}` expansion.

**Connection variables** (one per product, used by every task):

| Variable | Meaning |
|---|---|
| `opn_host` | OPNsense hostname or IP |
| `opn_port` | API port (default 443) |
| `opn_key` / `opn_secret` | API credentials |
| `opn_verify_ssl` | TLS verification toggle |

**Entity list variables** (when a playbook defines entities to ensure):

| Variable | Meaning |
|---|---|
| `opn_auth_users` | List of OPNsense users |
| `opn_auth_groups` | List of OPNsense groups |
| `opn_fw_aliases` | List of firewall aliases |
| `opn_fw_rules` | List of firewall rules |
| `opn_wg_servers` | List of WireGuard servers |

**Rules:**
- `{short}` is the product short prefix (`opn` for OPNsense). Each product picks its own short prefix once and keeps it consistent across library and collection.
- Entity list variables use **plural** (`users`, not `user`). Singular is used for a single-entity connection variable (`opn_host`).
- No dots, no hyphens, no camelCase. Flat snake_case only — enforced by Jinja/YAML parser behavior.

### 8. Test and playbook naming — mirror the source layout

**Python test files** mirror the `src/` structure — one test file per source file, same folder hierarchy.

```
src/opnsense/managers/auth/user.py         →  tests/unit/managers/auth/test_user.py
src/opnsense/managers/firewall/alias.py    →  tests/unit/managers/firewall/test_alias.py
```

| Kind | Location | Pattern | Example |
|---|---|---|---|
| Unit test | `tests/unit/<mirror>/` | `test_{entity}.py` | `tests/unit/managers/auth/test_user.py` |
| Integration test | `tests/integration/<domain>/` | `test_{entity}.py` | `tests/integration/auth/test_user.py` |

**Rules:**
- Unit tests **fully mirror** `src/` — agents and humans locate a test by the source path alone
- Integration tests are **flatter** (one level per domain) because they test end-to-end flows against a live device and don't need the same structural depth
- No `test_live_*` prefix — the `tests/integration/` folder already implies "live device"

**Ansible playbooks** are organized by **domain folder**, not by flat filenames.

```
playbooks/
├── auth/
│   ├── test_crud.yml
│   └── test_errors.yml
├── firewall/
│   ├── test_crud.yml
│   └── test_errors.yml
├── dns/
│   └── ...
└── test_all.yml              ← top-level aggregator only
```

| Location | File | Purpose |
|---|---|---|
| `playbooks/{domain}/test_crud.yml` | One per domain | CRUD / lifecycle playbook for all entities in that domain |
| `playbooks/{domain}/test_errors.yml` | One per domain | Error-path playbook (bad type, timeout, 401, etc.) |
| `playbooks/test_all.yml` | Top-level | Aggregator that imports every domain |

**Rules:**
- Domain folder names match §6 abbreviation table
- No flat `playbook_{domain}.yml` at the top level — domain folders are the structure
- One playbook file per test kind (CRUD, errors, lint, etc.) within each domain folder

### 9. Terraform resource naming

Pattern: `{type}_{product}_{purpose}` for module instances, `{product}-{purpose}` for resource `name` attributes.

| Context | Pattern | Example |
|---|---|---|
| Module source path | `modules/{product}-{purpose}` | `modules/proxmox-vm`, `modules/opnsense-alias` |
| Module instance | `module "{purpose}_{index}"` | `module "netbox_01" { source = "./modules/proxmox-vm" ... }` |
| Resource `name` attribute | `{product}-{purpose}-{seq:02d}` | `name = "vm-netbox-01"` (matches `naming/0001-infra`) |

Terraform state keys follow the module instance name. The `name` attribute of the created resource follows the infrastructure naming convention — so a Terraform-provisioned VM shows up in NetBox with the exact hostname pattern from `naming/0001-infra`.

### 10. API object naming — defer to other ADRs

Objects created through product APIs follow the ADR that owns that object type:

| Object type | ADR |
|---|---|
| Firewall aliases and rules | `naming/0003-firewall` |
| Identity (users, groups, service accounts) | `naming/0002-identity` |
| Infrastructure assets (hosts, VMs, service URLs) | `naming/0001-infra` |
| All other objects | `description` field follows `{ACTION} {context}` pattern (matches `naming/0003-firewall §4`) |

### 11. Revision triggers

Revise when:
- A new domain is added to §6 (new product or subsystem)
- Python library patterns diverge from `lib/python/0001-design-standard`
- A new repo type beyond the 5 in §1 is needed (unlikely — current set is exhaustive for the platform)
- Ansible collection structure changes (e.g. per-domain sub-collections rather than one per product)

## Consequences

- New repos follow the convention from day 1 — no naming debates
- Agents locate any module, class, test, or variable by applying the formula mechanically
- Domain abbreviation table is a single source of truth shared between Python libs, Ansible collections, and Terraform modules
- Ansible variable naming makes inventory files predictable
- Existing repos (`ansible-platform`, `lib-synology-dsm`) may require alignment on next refactor
- Short abbreviations (`ub`, `ts`, `rt`) trade discoverability for brevity — the table in §6 is the index
- Strict naming limits creative freedom — every name fits the pattern or the pattern is amended via ADR

## CISO mapping

| Framework | Controls covered |
|---|---|
| ISO 27001:2022 | A.8.9 (configuration management), A.8.25 (secure development lifecycle), A.8.28 (secure coding) |
