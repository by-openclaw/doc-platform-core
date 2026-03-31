# Naming & Identity Convention — DRAFT

> **Status:** DRAFT — evolves as platform layers land (Vault, Authentik, NetBox)
> **Date:** 2026-03-31
> **Authors:** @yboujraf + Claude Opus (brainstorm session)
> **Scope:** All envs, all repos, all services, all accounts
> **Revise when:** Layer 2 (Vault), Layer 3 (Identity/Authentik), Layer 5 (NetBox) are deployed
> **GRC tool:** CISO Assistant (open source) — compliance dashboard when deployed
>
> **Compliance frameworks:** This document maps to controls in:
> - **ISO 27001:2022** — Annex A (referenced as `ISO A.x.x.x`)
> - **NIS2** — Directive (EU) 2022/2555 (referenced as `NIS2 Art.xx`)
> - **GDPR** — Regulation (EU) 2016/679 (referenced as `GDPR Art.xx`)

---

## 1. Environments
> `ISO A.12.1.4` (separation of environments), `NIS2 Art.21(2)(a)` (risk management)

| Env | Full name | When to use |
|---|---|---|
| `poc` | Proof of concept | Lab / sandbox — pre-pipeline |
| `dev` | Development | Active development |
| `test` | Test | Automated testing |
| `staging` | Staging | Pre-prod validation |
| `acc` | Acceptance / UAT | Customer or stakeholder validation |
| `prod` | Production | Live — always labeled, never implicit |

**Rule: `prod` is always explicit.** No label = something is wrong, not "it's prod."

---

## 2. Environment label placement
> `ISO A.8.1.1` (asset inventory), `ISO A.12.1.4` (separation of environments)

Env label goes in the **same position** across all layers — consistency over convenience.

| Layer | Pattern | poc example | prod example |
|---|---|---|---|
| Hostname | `{function}-{type}-{env}-{seq:02d}` | `srv-proxmox-poc-01` | `srv-proxmox-prod-01` |
| VM | `vm-{service}-{env}-{seq:02d}` | `vm-netbox-poc-01` | `vm-netbox-prod-01` |
| FQDN | `{service}.{env}.{domain}` | `netbox.poc.by-systems.arpa` | `netbox.prod.by-systems.arpa` |
| Certificate | `*.{env}.{domain}` | `*.poc.by-systems.arpa` | `*.prod.by-systems.arpa` |

**Certificates:** One wildcard cert per env tier (`*.{env}.by-systems.arpa`). 6 env tiers = max 6 active certs. Issued by step-ca (Layer 2). No managed certs before step-ca is deployed — self-signed only until then.
| Secret file | `{scope}-{service}-{env}.json` | `infra-proxmox-poc.json` | `infra-proxmox-prod.json` |
| Vault path | `secret/{scope}/{service}/{env}` | `secret/infra/proxmox/poc` | `secret/infra/proxmox/prod` |
| Service account | `svc-{function}-{env}` | `svc-terraform-poc` | `svc-terraform-prod` |
| Group (env-scoped) | `grp-{env}-{role}` | `grp-poc-admin` | `grp-prod-admin` |
| NetBox tag | `{env}` | `poc` | `prod` |

---

## 3. Identity model — three account types
> `ISO A.9.2.1` (user registration/deregistration), `ISO A.9.2.2` (user access provisioning), `NIS2 Art.21(2)(i)` (human resources security)

### 3.1 Human accounts (one per person, all envs)

| Field | Convention |
|---|---|
| Standard | `{handle}` (e.g., `yboujraf`) |
| Admin | `adm_{handle}` (e.g., `adm_yboujraf`) |
| Per-env? | **No** — one identity, access controlled by group membership |
| Duplication | **Never** — same person, one standard + one admin account |

**Why two accounts per admin:**

| Account | Used for | Network | Privilege |
|---|---|---|---|
| `yboujraf` | Daily work — code, docs, dashboards, browse | Standard VLAN via SSO | Standard user |
| `adm_yboujraf` | Re-enable users, manage groups, SSH to infra, approve Terraform | OOB VLAN only (SSH 22222 + admin panels) | Privileged admin |

**Rule:** Never use `adm_*` for daily work. Never use standard account for admin tasks. Separation of concerns.

**Why underscore in `adm_`?** POSIX compatibility — when accounts sync to OS via LDAP/PAM, hyphens in usernames can cause issues on some systems. `adm_` is the safe prefix for OS-level accounts. All other naming uses hyphens (hostnames, services, secrets) — this is the one documented exception.

### 3.2 Service accounts (per-env, API only)

| Field | Convention |
|---|---|
| Naming | `svc-{function}-{env}` |
| Per-env? | **Yes** — separate credentials per env |
| Network | API only (no SSH, no SSO) |
| Privilege | Least privilege — only what the function needs |

| Account | Env | Purpose | Exact permissions |
|---|---|---|---|
| `svc-terraform-poc` | poc | Proxmox provisioning | Proxmox API: `VM.Allocate`, `VM.Config.*`, `VM.PowerMgmt`, `Datastore.*`, `SDN.Use` |
| `svc-terraform-prod` | prod | Same — different credentials | Same scope, different token |
| `svc-rune-poc` | poc | Ansible, lib-synology-dsm, NAS admin, monitoring | DSM API: `administrators` group (required for share/user/group CRUD). Authentik API: `user.write` only (re-enable accounts — no create, no group change). |
| `svc-rune-prod` | prod | Same — different credentials | Same scope, different token |

**Note:** `svc-rune-{env}` replaces the former `rune-api-{env}`. One service account per function per env. Permissions are enumerated — no broad admin "because it's easier."

### 3.3 Temporary accounts (per-env + expiry)

| Field | Convention |
|---|---|
| Naming | `tmp-{purpose}-{env}` |
| Per-env? | **Yes** |
| Expiry | **Mandatory** — set at creation, enforced by Authentik |
| Lifecycle | Auto-disabled on expiry. Reviewed quarterly. Never deleted (audit trail). |

| Account | Env | Purpose | Expiry |
|---|---|---|---|
| `tmp-contractor-alice-poc` | poc | External contractor | 2026-06-30 |
| `tmp-audit-ext-01-prod` | prod | External auditor | 2026-04-15 |

---

## 4. Groups — RBAC by environment
> `ISO A.9.1.1` (access control policy), `ISO A.9.2.3` (privileged access management), `NIS2 Art.21(2)(i)`

```
grp-{env}-{role}
```

| Group | Members | Access |
|---|---|---|
| `grp-poc-admin` | `adm_yboujraf` | Full admin on poc env |
| `grp-poc-readonly` | `yboujraf`, contractors | Read-only on poc |
| `grp-prod-admin` | `adm_yboujraf` (max 2-3 people) | Full admin on prod |
| `grp-prod-readonly` | `yboujraf`, monitoring | Read-only on prod |
| `grp-automation-poc` | `svc-terraform-poc`, `svc-rune-poc` | Automation access to poc (one account per function) |
| `grp-automation-prod` | `svc-terraform-prod`, `svc-rune-prod` | Automation access to prod (one account per function) |
| `grp-break-glass` | `adm_yboujraf` only | Emergency OOB access (password auth from OOB/MGMT) |

**Rule:** `grp-prod-admin` has max 2-3 members. If more people need prod admin → review the architecture, not the group.

---

## 5. Network segmentation enforces identity
> `ISO A.13.1.1` (network controls), `ISO A.13.1.3` (segregation in networks), `NIS2 Art.21(2)(a)` (risk management)

```
Standard VLAN:
  yboujraf → SSO → Grafana, GitLab, NetBox (standard access)
  tmp-contractor-alice-poc → SSO → poc resources only (group-scoped)

OOB VLAN (admin + automation only):
  adm_yboujraf → SSH 22222 → Proxmox, NAS, switches, firewalls
  svc-rune-poc → API → DSM, Proxmox API, Vault API

Standard users CANNOT reach OOB VLAN — network-level block.
```

---

## 6. Break-glass access
> `ISO A.9.4.4` (use of privileged utility programs), `ISO A.11.1.1` (physical security perimeter — OOB), `NIS2 Art.21(2)(c)` (business continuity)

| Who | How | When |
|---|---|---|
| `adm_yboujraf` (member of `grp-break-glass`) | Password auth + SSH key from OOB/MGMT VLAN only | Authentik down, SSH key lost, everything broken |

Break-glass is the last resort. It bypasses SSO. It works when everything else fails. Only `adm_*` accounts in `grp-break-glass` — never service accounts.

---

## 7. Automation re-enabling accounts
> `ISO A.9.2.5` (review of user access rights), `ISO A.12.4.1` (event logging), `NIS2 Art.21(2)(b)` (incident handling)

```
User locked out
  → Monitoring detects (heartbeat or alert)
    → svc-rune-poc calls Authentik API → re-enable account
      → Authentik permission scope: user.write ONLY
        - ✅ Can: re-enable disabled accounts
        - ❌ Cannot: create accounts
        - ❌ Cannot: change group membership
        - ❌ Cannot: modify admin accounts (adm_*)
      → Audit log: which svc re-enabled which user, when
        → adm_yboujraf reviews in next session
          → Discord notification to #bot-openclaw
```

**Note:** `svc-rune-poc` for poc env, `svc-rune-prod` for prod. Same scope, different credentials. The Authentik API token is env-scoped — a poc token cannot re-enable prod accounts.

---

## 8. Credential storage — per type
> `ISO A.10.1.1` (cryptographic controls policy), `ISO A.9.4.3` (password management), `NIS2 Art.21(2)(d)` (supply chain security)

| Account type | Credential stored where | Secret file naming |
|---|---|---|
| Human (standard) | Personal Vaultwarden | No file — personal vault |
| Human (admin) | Personal Vaultwarden + OOB SSH key | No file — personal vault |
| Service (per-env) | `workspace/infra/secrets/` JSON → Vault Phase 2 | `{scope}-{service}-{env}.json` |
| Temporary | Authentik-managed (invite flow) | No file — Authentik handles lifecycle |

---

## 9. Lifecycle & purge
> `ISO A.9.2.6` (removal of access rights), `ISO A.9.2.5` (review of user access rights), `GDPR Art.17` (right to erasure), `GDPR Art.32` (security of processing), `NIS2 Art.21(2)(i)` (HR security)

| Account type | Created by | Expiry | Disabled by | Purge |
|---|---|---|---|---|
| Human | @yboujraf (Authentik or manual) | Never (tied to employment) | Manual on offboarding | Disable, never delete |
| Human admin | @yboujraf | Never | Manual on offboarding | Disable, never delete |
| Service (permanent) | Ansible / Terraform | Never | Credential rotation (Vault) | Rotate credential, keep account |
| Temporary | @yboujraf (invite-only) | Fixed date at creation | Authentik auto-disable on expiry | Quarterly review, keep disabled for audit |

**NIS2 / ISO 27001 / GDPR alignment:**

| Requirement | How we handle it |
|---|---|
| NIS2 Art. 21 — access management | Env isolation, group-based RBAC, separate admin accounts |
| ISO 27001 A.9.2.6 — removal of access | Auto-disable on expiry (Authentik). Disable, never delete. |
| ISO 27001 A.9.2.5 — access review | Quarterly review of all disabled/expired accounts |
| GDPR Art. 17 — right to erasure | Anonymize on explicit request + legal review. Default = disabled. |
| ISO 27001 A.9.4 — audit trail | Full history preserved: logins, group changes, enable/disable events |

---

## 10. Revision triggers
> `ISO A.18.2.1` (independent review of information security), `ISO A.5.1.2` (review of policies)

Revise this document when:

| Event | What to revise |
|---|---|
| Vault deployed (Layer 2) | Credential storage section — local JSON → Vault |
| Authentik deployed (Layer 3) | Account lifecycle, auto-disable, SSO integration |
| NetBox deployed (Layer 5) | Add NetBox modeling (env tags, contact types, account inventory) |
| First prod environment | Validate prod naming, prod groups, prod access controls |
| First external user / contractor | Validate temp account flow end-to-end: invite, expiry enforcement, Authentik auto-disable, quarterly review process |
| NIS2/ISO audit preparation | Review all sections against audit checklist |

---

*DRAFT — review with team after each layer deployment. This is the starting point, not the final state.*
