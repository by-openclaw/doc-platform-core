# ADR-0010: Naming & Identity Convention

**Status:** Accepted
**Date:** 2026-03-31
**Deciders:** @yboujraf

## Context

The BY-SYSTEMS platform spans multiple environments, services, and account types. Without a binding naming convention, each layer invents its own patterns — hostnames diverge from Vault paths, service accounts are named inconsistently, and environment labels are implicit or missing. This creates operational risk: a credential for `poc` could be mistakenly applied to `prod` if naming doesn't enforce separation.

The naming & identity convention draft (`docs/naming-and-identity-convention-draft.md`) was developed in a brainstorm session and refined with compliance tags. This ADR promotes that draft to a binding architectural decision.

## Decision

### 1. Environment tiers — explicit, ordered, no implicit prod

Six environment tiers, always explicit:

| Env | Full name | When to use |
|---|---|---|
| `poc` | Proof of concept | Lab / sandbox — pre-pipeline |
| `dev` | Development | Active development |
| `test` | Test | Automated testing |
| `staging` | Staging | Pre-prod validation |
| `acc` | Acceptance / UAT | Customer or stakeholder validation |
| `prod` | Production | Live — always labeled, never implicit |

**Rule:** `prod` is always explicit. No label = something is wrong, not "it's prod."

### 2. Environment label placement — consistent position across all layers

| Layer | Pattern | Example (poc) | Example (prod) |
|---|---|---|---|
| Hostname | `{function}-{type}-{env}-{seq:02d}` | `srv-proxmox-poc-01` | `srv-proxmox-prod-01` |
| VM | `vm-{service}-{env}-{seq:02d}` | `vm-netbox-poc-01` | `vm-netbox-prod-01` |
| LXC | `lxc-{service}-{env}-{seq:02d}` | `lxc-pihole-poc-01` | `lxc-pihole-prod-01` |
| FQDN | `{service}.{env}.{domain}` | `netbox.poc.by-systems.be` | `netbox.prod.by-systems.be` |
| Certificate | `*.{env}.{domain}` | `*.poc.by-systems.be` | `*.prod.by-systems.be` |
| Secret file | `{scope}-{service}-{env}.json` | `infra-proxmox-poc.json` | `infra-proxmox-prod.json` |
| Vault path | `secret/{scope}/{service}/{env}` | `secret/infra/proxmox/poc` | `secret/infra/proxmox/prod` |
| Service account | `svc-{function}-{env}` | `svc-terraform-poc` | `svc-terraform-prod` |
| Group (env-scoped) | `grp-{env}-{role}` | `grp-poc-admin` | `grp-prod-admin` |
| NetBox tag | `{env}` | `poc` | `prod` |

### 3. Three account types

**Human accounts** — one per person, all envs:
- Standard: `{handle}` (e.g., `yboujraf`)
- Admin: `adm_{handle}` (e.g., `adm_yboujraf`)
- Per-env: **No** — one identity, access controlled by group membership
- **Why underscore in `adm_`?** POSIX compatibility — hyphens in usernames cause issues when accounts sync to OS via LDAP/PAM. All other naming uses hyphens; this is the one documented exception.
- **Rule:** Never use `adm_*` for daily work. Never use standard account for admin tasks.

**Service accounts** — per-env, API only:
- Naming: `svc-{function}-{env}`
- Per-env: **Yes** — separate credentials per environment
- Privilege: least privilege — only what the function needs, permissions enumerated

**Temporary accounts** — per-env + mandatory expiry:
- Naming: `tmp-{purpose}-{env}`
- Expiry: mandatory, set at creation, enforced by Authentik
- Lifecycle: auto-disabled on expiry, reviewed quarterly, never deleted (audit trail)

### 4. Groups — RBAC by environment

Pattern: `grp-{env}-{role}`

| Group | Members | Access |
|---|---|---|
| `grp-{env}-admin` | `adm_*` accounts (max 2-3 for prod) | Full admin on env |
| `grp-{env}-readonly` | Standard accounts, contractors | Read-only on env |
| `grp-automation-{env}` | `svc-*` accounts | Automation access to env |
| `grp-break-glass` | `adm_yboujraf` only | Emergency OOB access (password auth from OOB/MGMT) |

**Rule:** `grp-prod-admin` has max 2-3 members. If more people need prod admin, review the architecture.

### 5. Network segmentation enforces identity

- Standard VLAN: standard accounts via SSO to Grafana, GitLab, NetBox
- OOB VLAN: admin + automation accounts only (SSH 22222, API access)
- Standard users cannot reach OOB VLAN — network-level block

### 6. Break-glass access

Only `adm_*` accounts in `grp-break-glass`. Password auth + SSH key from OOB/MGMT VLAN only. Bypasses SSO. Works when everything else fails. Never service accounts.

### 7. Credential storage per account type

| Account type | Credential stored where |
|---|---|
| Human (standard) | Personal Vaultwarden |
| Human (admin) | Personal Vaultwarden + OOB SSH key |
| Service (per-env) | `workspace/infra/secrets/` JSON → Vault (Phase 2) |
| Temporary | Authentik-managed (invite flow) |

### 8. Lifecycle & purge

| Account type | Expiry | Disabled by | Purge |
|---|---|---|---|
| Human | Never (tied to employment) | Manual on offboarding | Disable, never delete |
| Human admin | Never | Manual on offboarding | Disable, never delete |
| Service (permanent) | Never | Credential rotation (Vault) | Rotate credential, keep account |
| Temporary | Fixed date at creation | Authentik auto-disable | Quarterly review, keep disabled for audit |

### 9. Revision triggers

Revise this ADR when: Vault deployed (Layer 2), Authentik deployed (Layer 3), NetBox deployed (Layer 5), first prod environment, first external user/contractor, NIS2/ISO audit preparation.

## Enforcement

| Repo | Mechanism | File |
|---|---|---|
| ansible-platform | CI workflow — checks `env:` declaration in group_vars, blocks `env: prod` in non-prod inventories, warns on service account names missing env tier | `.github/workflows/naming-check.yml` |

Until Authentik (Layer 3) provides policy-as-code, naming is enforced by CI checks and code review.

## Consequences

**Positive:**
- Environment isolation is enforced at every layer — naming makes cross-env mistakes visible
- Account types are unambiguous — `svc-*` is automation, `adm_*` is privileged, `tmp-*` expires
- Group membership is the single access control mechanism — no per-resource ACLs to maintain
- Credential separation per env prevents blast radius expansion (poc token cannot touch prod)
- Audit trail preserved — accounts disabled, never deleted

**Negative:**
- `adm_` underscore prefix is inconsistent with the hyphen convention everywhere else (necessary for POSIX compatibility)
- Per-env service accounts multiply credential count (6 envs x N functions)
- Break-glass access bypasses SSO — must be tightly controlled and audited
- Convention must be manually enforced until Authentik (Layer 3) provides policy-as-code

## Compliance

- **ISO A.8.1.1** (asset inventory) — consistent naming enables automated inventory
- **ISO A.9.1.1** (access control policy) — group-based RBAC with env scoping
- **ISO A.9.2.1** (user registration/deregistration) — three account types with defined lifecycle
- **ISO A.9.2.2** (user access provisioning) — least privilege, permissions enumerated per service account
- **ISO A.9.2.3** (privileged access management) — separate admin accounts (`adm_*`), max 2-3 prod admins
- **ISO A.9.2.5** (review of user access rights) — quarterly review of disabled/expired accounts
- **ISO A.9.2.6** (removal of access rights) — auto-disable on expiry, disable never delete
- **ISO A.9.4.4** (use of privileged utility programs) — break-glass restricted to OOB VLAN
- **ISO A.12.1.4** (separation of environments) — 6 explicit tiers, env label in every naming layer
- **ISO A.13.1.1** (network controls) — standard vs OOB VLAN separation
- **ISO A.13.1.3** (segregation in networks) — admin/automation on OOB only
- **NIS2 Art.21(2)(a)** (risk management) — env separation reduces blast radius
- **NIS2 Art.21(2)(b)** (incident handling) — automation re-enable with scoped permissions + audit log
- **NIS2 Art.21(2)(c)** (business continuity) — break-glass access as last resort
- **NIS2 Art.21(2)(i)** (human resources security) — account lifecycle tied to employment/contract
- **GDPR Art.17** (right to erasure) — anonymize on explicit request + legal review; default = disabled
- **GDPR Art.32** (security of processing) — credential separation, least privilege, audit trail

## Notes

- Source: `docs/naming-and-identity-convention-draft.md` (promoted to this ADR)
- Environment tier standard extracted to ADR-0012 for independent referenceability
- Credential storage convention extracted to ADR-0011
- GRC tool: CISO Assistant (open source) — compliance dashboard when deployed
- Domain: `by-systems.arpa` was original placeholder. Decision 2026-04-01: use `{service}.{env}.by-systems.be` with split DNS.
