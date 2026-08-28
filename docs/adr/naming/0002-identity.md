# naming/0002 — Identity Naming

**Status:** Draft
**Date:** 2026-04-13 (supersedes part of flat ADR-0010, 2026-03-31)
**Scope:** Patterns for account and group *names*. Does not define lifecycle, authentication, or authorization.
**Related:** `identity/0001-authentication`, `identity/0002-provisioning`, `identity/0004-os-accounts`, `naming/0001-infra`

---

## Context

Account and group names must be unambiguous, stable, and mechanically parseable. Without a binding pattern, human accounts, service accounts, and group memberships drift per tool — `yboujraf` here, `y.boujraf` there, `rune-ci` vs `svc-rune` vs `ci-rune`.

This ADR defines **the patterns only**. Account lifecycle (creation, key rotation, sudo, disable-never-delete) is in `identity/0004-os-accounts`. Provisioning from Authentik to downstream tools is in `identity/0002-provisioning`.

## Decision

### 1. Three account categories

| Category | Purpose | Env-scoped? |
|---|---|---|
| **Human** | One person, multiple systems | No — one identity, access by group |
| **Service** | Automation, API, CI/CD | **Yes** — per-env credentials, scope-bound |
| **Temporary** | Contractor, short-lived, PoC access | Yes + mandatory expiry |

Lifecycle rules (disable-never-delete, quarterly review, OOB break-glass) are in `identity/0004-os-accounts`.

### 2. Human account pattern

| Kind | Pattern | Example |
|---|---|---|
| Standard | `{handle}` | `yboujraf` |
| Admin | `adm_{handle}` | `adm_yboujraf` |

**Rules:**
- One handle per person, chosen at onboarding, immutable thereafter
- Handle is lowercase, `[a-z0-9]+`, no hyphens or dots
- **Admin prefix uses underscore** — `adm_` — this is the documented POSIX compatibility exception. Every other identifier on the platform uses hyphens; human admin accounts are the exception because LDAP/PAM hyphenated usernames break on some Linux distributions
- Admin account is for admin tasks only. Daily work uses the standard account. Never mix.
- Access scope is controlled by group membership, not by the account name
- Matching email: `{handle}@by-systems.be` for standard, `adm_{handle}@by-systems.be` for admin

### 3. Service account pattern

```
svc-{function}-{env}
```

Examples:

| Account | Function | Env |
|---|---|---|
| `svc-terraform-prod` | Terraform runs | prod |
| `svc-terraform-dev` | Terraform runs | dev |
| `svc-ansible-prod` | Ansible runs | prod |
| `svc-rune-prod` | Rune ops automation | prod |
| `svc-vault-agent-prod` | Vault agent sidecar | prod |
| `svc-gitlab-runner-prod` | GitLab CI runner | prod |

**Rules:**
- **Env is part of the name** — this is the exception to the "no env in name" rule from `naming/0001-infra`. Rationale: a service account's credentials are scope-bound to one env. Moving credentials across envs is not a rename, it's a new account. The env token is immutable for the life of the credential.
- One account per (function, env) pair. Never share a credential across envs.
- Function token is short, descriptive, lowercase, hyphen-separated
- Full pattern is lowercase hyphen-separated
- Service accounts never have admin prefixes — they are not humans
- Credentials are stored in HashiCorp Vault under `secret/{env}/{tool}/...` (see `security/0001-secret-storage` — future, from flat 0011+0016)

### 4. Temporary account pattern

```
tmp-{purpose}-{env}
```

Examples:

| Account | Purpose | Env |
|---|---|---|
| `tmp-audit-nis2-prod` | NIS2 auditor read-only access | prod |
| `tmp-migration-mailcow-dev` | One-off Mailcow data migration | dev |
| `tmp-contractor-acme-dev` | Acme Corp contractor | dev |

**Rules:**
- Mandatory expiry set at creation (enforced by Authentik)
- Purpose token is specific — not generic like `tmp-test-prod`
- Env is part of the name (same rationale as service accounts)
- Auto-disabled on expiry, reviewed quarterly, never deleted (audit trail per `identity/0004-os-accounts`)

### 5. OS / Linux group pattern

Linux groups are **bare names** (no prefix), matching Linux convention (`sudo`, `wheel`, `adm`, `docker`). They are created by the Ansible `user-mgmt` / `hardening` roles or by stock `useradd`, and referenced directly by sshd, PAM, sudoers, and Docker — the names must match what the roles and system configs expect.

There are two kinds of Linux groups on this platform, and they are named differently:

> **`{org}` — deployment variable.** The deployment's organisation short-name, resolved from the domain per [`naming/0001-infra §9`](0001-infra.md): **`by-research`** for the `by-research.be` deployment (a future client `client-xyz.com` → `client-xyz`). The **local break-glass admin account and its primary group are named `{org}`** — a placeholder resolved from the deployment manifest, exactly like `{domain}`/`{env}`, **never a hardcoded literal.** (Distinct from the human-admin email domain `@by-systems.be` in §2, which is the operator's own domain.)

#### 5.1 User primary groups — created with each account

Every user account has a primary group with the **same name as the user**, auto-created by `useradd` (Linux default). The SSH-allowed list in sshd hardening references these primary groups directly.

| Group | Created with user | Auth method | In sshd `AllowGroups`? |
|---|---|---|---|
| `{org}` | **Local break-glass** (always in `/etc/passwd`, never Authentik) | Key from anywhere + **password from OOB CIDR** (via `break-glass` role group, see §5.2) | ✅ (default) |
| `rune` | `svc-rune-{env}` — interactive service account (human at keyboard via Rune agent) | Key only | ✅ (default) |
| `ansible` | `svc-ansible-{env}` — non-interactive automation account (playbooks, CI) | Key only + `ansible_become_pass` from Vault | ⏸ will be added when that account is provisioned (planned, post-Vault) |
| `adm_yboujraf` | Human admin account (Authentik when live, local fallback) | Key only | Added on hosts where admin presence is required |
| `yboujraf` | Standard human account | Key only | Added on dev VMs only |

These names are **not a naming convention** — they are mechanical consequences of account creation (`useradd foo` creates `foo:foo`). Adding a new user that should SSH means adding the corresponding primary group to `hardening_ssh_allow_groups`.

**Critical — only `{org}` can use password auth.** All other accounts (`svc-rune-*`, `svc-ansible-*`, `adm_*`, standard humans) are **key-only, always**, regardless of source network. The OOB password fallback is reserved exclusively for the local-only break-glass identity. This is enforced by the `break-glass` role group membership (§5.2), which contains only `{org}`.

**Passphrase-less SSH keys are forbidden — all credentials via Vault:**

Every SSH private key on the platform **must** have a passphrase. Empty-passphrase keys are not allowed, anywhere, for any account. This is enforced per `identity/0004-os-accounts §2 SSH keys`.

The passphrase itself is never stored on disk and never carried in the account. It is retrieved at session start from HashiCorp Vault:

| Account | Passphrase source | Retrieved by |
|---|---|---|
| `{org}` | Operator memory (primary) + Vault KV (backup) | Typed at console / SSH during break-glass |
| `svc-rune-{env}` | **Vault KV** (`secret/{env}/svc-rune/ssh-passphrase`) | Rune agent at session start — `ssh-add` with Vault-sourced passphrase |
| `svc-ansible-{env}` | **Vault KV** (`secret/{env}/svc-ansible/ssh-passphrase`) | Ansible runtime — `ssh-agent` populated via Vault lookup |
| `adm_yboujraf` | Operator memory + personal Vaultwarden (see §2 note on Vaultwarden vs HashiCorp Vault) | Typed by human at session start |
| `yboujraf` | Operator memory + personal Vaultwarden | Typed by human at session start |

**Consequences:**

- No `svc-*` account has a usable key without access to HashiCorp Vault. Vault compromise = key passphrase compromise.
- Vault access for passphrase retrieval uses per-env AppRole / policy scopes — `svc-rune-dev` cannot read `svc-rune-prod` secrets.
- Key rotation = rotate key **and** passphrase **and** update Vault record, all together.
- Credential storage paths are defined in `security/0001-secret-storage` (future, from flat 0011+0016).

**svc-rune vs svc-ansible — separation of privilege:**

Both use Vault-backed passphrases. The distinction is operational, not cryptographic:

| | `svc-rune-{env}` | `svc-ansible-{env}` |
|---|---|---|
| Session trigger | Human initiates (via Rune agent) | Playbook / CI runs unattended |
| Passphrase retrieval | Vault, at session start | Vault, at playbook-run start |
| `sudo` password | Vault (`svc-rune` policy) | Vault (`svc-ansible` policy) — injected as `ansible_become_pass` |
| `docker` group | ✅ on container hosts | ❌ |
| Blast radius | Interactive ops | Automation-scoped (what playbooks can do) |

If either credential leaks, the attacker gains only that account's Vault scope — `svc-rune-dev` compromise does not unlock `svc-rune-prod`, nor `svc-ansible-*`, nor break-glass. **No passphrase-less key exists anywhere on the platform to exploit.**

#### 5.2 Role groups — explicitly created for a function

Role groups are created by Ansible, not by `useradd`. They represent capabilities, not users.

| Group | Members | Purpose | Referenced by |
|---|---|---|---|
| **`break-glass`** | `{org}` only | **SSH gate AND password-auth fallback from OOB CIDR** — both via one group | `ansible-platform/roles/hardening/templates/sshd_config.j2` — appears in `AllowGroups` **and** in the `Match Group ... Address <oob-cidr>` block |
| `sudo` | Selected users per host | Sudo access (standard Linux, sudoers) | PAM, `/etc/sudoers.d/*` |
| `docker` | `svc-rune` on container hosts only | Docker ops — root-equivalent, restricted per `identity/0004 §7` | Docker daemon |

#### 5.3 `break-glass` is authoritative — one group, two jobs

The group `break-glass` is **unified** — a single Linux group that sshd hardening uses **both** as an `AllowGroups` entry **and** as the `Match Group` for password-auth fallback from the OOB CIDR.

Result:
- Any `break-glass` member can SSH from anywhere allowed by `AllowGroups`
- From the OOB management network **additionally**, password authentication is accepted
- There is no separate "ssh-allowed" group distinct from "break-glass" — the role group IS the ssh gate for break-glass identities

The token `break-glass` must stay synchronized in three places:

1. `ansible-platform/roles/hardening/templates/sshd_config.j2` (the template)
2. `ansible-platform/roles/hardening/defaults/main.yml` — `hardening_ssh_break_glass_group: "break-glass"`
3. `identity/0004-os-accounts §5 SSH break-glass`
4. This naming ADR

Renaming `break-glass` requires coordinated updates to all four. The Ansible default must not change without an ADR amendment.

**Rules:**
- Linux groups are **env-agnostic** — the host already carries env context via NetBox and DNS zone
- Linux groups use **bare names**, no `grp-` prefix
- Role groups (§5.2) are created and populated by the Ansible `user-mgmt` / `hardening` roles, never by ad-hoc `gpasswd`
- Primary groups (§5.1) are created automatically by `useradd` via the same roles — no manual intervention
- Do not create new role groups outside §5.2 without an ADR amendment

### 5.1 Platform RBAC groups (LDAP / Authentik — future)

When Authentik deploys with LDAP provider, **platform RBAC groups** (abstract, cross-host, env-scoped) may be introduced alongside the bare Linux groups. These are a distinct namespace and use the `grp-` prefix to avoid collision with Linux system groups.

```
grp-{env}-{role}
```

Example future groups:

| Group | Purpose |
|---|---|
| `grp-prod-dbadmin` | Database admins, prod env only |
| `grp-dev-netadmin` | Network admins, dev env only |

**Rules for platform RBAC groups:**
- `grp-` prefix distinguishes them from Linux system groups (§5)
- Env token is part of the name — scope is explicit
- Deferred until Authentik is deployed. Until then, access control uses the bare Linux groups in §5 plus Authentik-platform groups in §6.

### 6. Authentik platform group pattern

Authentik groups are a **separate namespace** from OS groups. They govern access to platform tools (GitLab, NetBox, Grafana, Vault, …) via the identity-sync framework (`identity/0002-provisioning`).

```
{prefix}{scope}-{role}
```

| Prefix | Meaning | Example |
|---|---|---|
| `tool-` | Access to a specific platform tool | `tool-gitlab-developers`, `tool-netbox-admins`, `tool-grafana-editors` |
| `svc-` | Service account grouping (not to be confused with the `svc-*` account pattern from §3) | `svc-ansible-sync`, `svc-vault-agents` |
| `prj-` | Project-scoped access (contractors, client projects) | `prj-client-xyz-developers`, `prj-internal-audit-readonly` |
| `org-` | Org-wide platform roles | `org-admins`, `org-readonly`, `org-engineers` |

**Rules:**
- All lowercase, hyphen-separated. No uppercase. No underscores. (The `adm_` underscore exception from §2 is human-account-only.)
- Groups describe **intent**, not target-platform mechanics. One Authentik group may map to different roles in different tools — the mapping lives in the identity-sync vars files, not in the group name.
- Authentik group names are **env-agnostic**. Env scoping is expressed in the target tool's mapping (Ansible vars file per tool). Rationale: the same person should be a "developer" across envs for a given tool; access control per env is enforced by the tool, not by Authentik group membership.
- Ad-hoc groups created for one-off needs are not part of the standard convention unless they recur and can be generalized.

### 7. Cross-reference table

| Concern | Lives in |
|---|---|
| Account creation, SSH/GPG keys, sudo, shells | `identity/0004-os-accounts` |
| Authentication flow (OIDC, SAML, fallback) | `identity/0001-authentication` |
| How groups map to tool roles | `identity/0002-provisioning` |
| Machine credentials (CI tokens) | `identity/0003-machine-credentials` |
| Credential storage (Vault paths) | `security/0001-secret-storage` (future) |
| OS hardening, break-glass CIDR | `identity/0004-os-accounts §5` |

### 8. Revision triggers

Revise when:
- A new account category is needed (e.g. robot/agent accounts — currently treated as `svc-*`)
- Authentik replaces the current group namespace with something not prefix-based
- A new platform tool requires a group-prefix token that doesn't fit the 4 current ones
- The `adm_` POSIX exception is no longer needed (e.g. all downstream LDAP consumers accept hyphens)

## Consequences

- Every account name carries its category prefix and (for service/temp) env token
- Per-env service accounts multiply credential count (N functions × M envs), accepted because it isolates blast radius
- OS groups and Authentik groups coexist as two disjoint namespaces — no name collision possible
- `adm_` underscore is the single documented exception to the hyphen convention; everywhere else uses hyphens
- Adding a new tool to Authentik = pick a `tool-{name}-{role}` group, map in identity-sync vars — no ADR change
- Naming alone enforces no lifecycle rules. Lifecycle is enforced by `identity/0004-os-accounts` + Ansible `user-mgmt` role.

## CISO mapping

| Framework | Controls covered |
|---|---|
| ISO 27001:2022 | A.5.15 (access control), A.5.16 (identity management), A.8.2 (privileged access rights — `adm_` separation) |
| NIS2 | Art. 21(2)(i) (human resources security — naming makes admin-vs-standard distinction visible) |
