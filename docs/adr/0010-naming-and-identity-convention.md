# ADR-0010: Naming & Identity Convention

**Status:** Accepted
**Date:** 2026-03-31
**Updated:** 2026-04-03 — prod env omitted from hostnames/VM names; domain examples updated to by-research.be
**Deciders:** @yboujraf

## Context

The BY-SYSTEMS platform spans multiple environments, services, and account types. Without a binding naming convention, each layer invents its own patterns — hostnames diverge from Vault paths, service accounts are named inconsistently, and environment labels are implicit or missing. This creates operational risk: a credential for `poc` could be mistakenly applied to `prod` if naming doesn't enforce separation.

The naming & identity convention draft (`docs/naming-and-identity-convention-draft.md`) was developed in a brainstorm session and refined with compliance tags. This ADR promotes that draft to a binding architectural decision.

## Decision

### 1. Environment tiers — explicit, ordered, no implicit prod

Six environment tiers, always explicit:

| Env | Full name | When to use |
|---|---|---|
| `poc` | Proof of concept | Interoperability validation — new hardware, network config, multi-component concept testing. Used before writing project plan. Not used for single-library work. |
| `dev` | Development | Active development |
| `test` | Test | Automated testing |
| `staging` | Staging | Pre-prod validation |
| `acc` | Acceptance / UAT | Customer or stakeholder validation |
| `prod` | Production | Live — always labeled, never implicit |

**Rule:** `prod` is always labeled in metadata, tags, secret paths, and deployment manifests. However, `prod` is **omitted from the hostname and VM name** — the prod tag on the asset is the environment marker. All non-prod tiers (`poc`, `dev`, `test`, `staging`, `acc`) carry `{env}` explicitly in the name.

> **Rationale:** Prod hostnames are clean and short (`vm-netbox-01`). Non-prod carries env to prevent cross-env confusion. The prod tag in NetBox, Proxmox, secret file paths, and Vault paths still carries `prod` explicitly.

### 2. Environment label placement — consistent position across all layers

| Layer | Pattern | Example (poc) | Example (prod) |
|---|---|---|---|
| Hostname | `{function}-{type}-{env}-{seq:02d}` | `srv-proxmox-poc-01` | `srv-proxmox-01` |
| VM | `vm-{service}-{env}-{seq:02d}` | `vm-netbox-poc-01` | `vm-netbox-01` |
| LXC | `lxc-{service}-{env}-{seq:02d}` | `lxc-pihole-poc-01` | `lxc-pihole-01` |
| VM FQDN | `{hostname}.{domain}` | `vm-netbox-poc-01.{domain}` | `vm-netbox-01.{domain}` |
| Service URL | `{alias}.{domain}` (config) | `netbox.poc.{domain}` (defined in Traefik) | `netbox.{domain}` (defined in Traefik) |
| Certificate | `*.{domain}` | `*.{domain}` (see ADR-0014 for CA selection) | `*.{domain}` (see ADR-0014 for CA selection) |
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

### 9. Authentik platform group naming (additive namespace)

Authentik groups for platform tool access follow a separate naming convention from OS/infra groups. These are two distinct namespaces — no collision, no replacement.

**OS/infra groups** (Proxmox, LDAP, OS — existing):
```
grp-admins       ← OS/Proxmox admin group (env-agnostic — the machine hostname is the env context)
grp-readonly     ← OS read-only group
grp-ops          ← Ops/automation accounts
grp-break-glass  ← Emergency OOB access only
```

Note: OS group names do NOT include env prefix. The VM hostname already carries env context (`vm-netbox-poc-01`). One group name, deployed on env-specific machines.

**Authentik platform groups** (new, Authentik-managed):

| Prefix | Meaning | Example |
|---|---|---|
| `tool-` | Access to a specific platform tool | `tool-gitlab-developers`, `tool-netbox-admins` |
| `svc-` | Service account groups | `svc-ansible-sync`, `svc-vault-agent` |
| `prj-` | Project-scoped groups (contractors, specific repos) | `prj-client-xyz-developers` |
| `org-` | Org-wide platform roles | `org-admins`, `org-readonly` |

Full pattern: `{prefix}{tool}-{role}` for tool groups, `{prefix}{project}-{role}` for project groups.
All lowercase, hyphen-separated. No uppercase, no underscores.

These groups are defined and managed in Authentik. They map to tool-specific roles via the identity-sync Ansible framework (ADR-0024).

Ad-hoc groups (e.g., for a specific PoC project) are created on demand — not part of the standard convention unless they recur.

### 10. `{domain}` — deployment variable

VM FQDNs, service URLs, and certificates use `{domain}` as a placeholder. The actual domain is set per deployment in the deployment manifest:

```yaml
# deployments/{org}-{env}/deployment.yml
org:
  domain: by-systems.be   # resolves {domain} for this deployment
  env: prod
```

Examples of how `{domain}` resolves:

| Deployment | `{domain}` | VM FQDN example | Service URL example |
|---|---|---|---|
| BY-SYSTEMS poc | `by-research.be` | `vm-netbox-poc-01.by-research.be` | `netbox.poc.by-research.be` |
| BY-SYSTEMS prod | `by-research.be` | `vm-netbox-01.by-research.be` | `netbox.by-research.be` |
| Client XYZ prod | `client-xyz.com` | `vm-netbox-01.client-xyz.com` | `netbox.client-xyz.com` |

No ADR changes required when a domain changes — update the deployment manifest only.

### 11. Revision triggers


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

## CISO mapping

> Applies only to controls directly relevant to this ADR's scope.
> Do NOT list every ISO control — only those this ADR satisfies, partially satisfies, or gaps.

### ISO/IEC 27001:2022

| Control | Title | Status | Notes |
|---|---|---|---|
| A.5.15 | Access control | ✓ Covered | Group-based RBAC with env scoping defined |
| A.5.16 | Identity management | ✓ Covered | Three account types with defined lifecycle |
| A.5.17 | Authentication information | ✓ Covered | Per-env service account credentials; break-glass controls defined |
| A.5.18 | Access rights | ✓ Covered | Least privilege; permissions enumerated per service account |
| A.8.2 | Privileged access rights | ✓ Covered | Separate `adm_*` accounts; max 2-3 prod admins; quarterly review |
| A.8.5 | Secure authentication | ✓ Covered | Break-glass restricted to OOB VLAN; SSO via Authentik for standard users |
| A.8.18 | Use of privileged utility programs | ✓ Covered | Admin accounts restricted to OOB VLAN |
| A.8.1 | User end point devices | ⚠ Partial | Asset inventory convention defined; NetBox not yet deployed |

### NIS2 (Directive 2022/2555)

| Article | Requirement | Status | Notes |
|---|---|---|---|
| Art. 21(2)(a) | Risk management policies | ✓ Covered | Env separation reduces blast radius; naming convention enforces it |
| Art. 21(2)(b) | Incident handling | ✓ Covered | Scoped automation accounts with env isolation limit incident scope |
| Art. 21(2)(i) | Human resources security | ✓ Covered | Account lifecycle tied to employment; disable never delete |

### GDPR (Regulation 2016/679)

| Article | Requirement | Status | Notes |
|---|---|---|---|
| Art. 17 | Right to erasure | ✓ Covered | Anonymise on explicit request + legal review; default = disabled (audit trail preserved) |
| Art. 32 | Security of processing | ✓ Covered | Credential separation, least privilege, audit trail via account lifecycle rules |

## Notes

- Source: `docs/naming-and-identity-convention-draft.md` (promoted to this ADR)
- Environment tier standard independently defined in a separate ADR for independent referenceability
- Credential storage convention independently defined in a separate ADR
- GRC tool: CISO Assistant (open source) — compliance dashboard when deployed
- **DNS domain:** VM FQDNs use `{hostname}.{domain}` (forward DNS). `.arpa` is used only for reverse DNS (PTR records) — NOT for forward-facing service or VM FQDNs. Service URLs are defined in Traefik config; split DNS is handled via Pi-hole/Unbound internally. Wildcard cert: `*.{domain}` — CA selection per ADR-0014.
- **`{domain}` resolution:** Set per deployment in `deployments/{org}-{env}/deployment.yml`. See section 10.
