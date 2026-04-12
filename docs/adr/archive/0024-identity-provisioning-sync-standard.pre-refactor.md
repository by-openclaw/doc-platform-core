# ADR-0024: Identity Provisioning & Sync Standard

**Date:** 2026-04-02
**Status:** Accepted
**Deciders:** yboujraf, Rune
**Extends:** ADR-0004 (SSO/authentication), ADR-0010 (naming), ADR-0016 (Vault paths)

---

## Context

ADR-0004 defines how users **authenticate** to the platform (Authentik as IAM hub, optional upstream federation). Authentication is not the same as provisioning. Authentik SSO tells a tool who the user is — it does not automatically create the user inside the tool, assign them to the correct group, or apply the correct role.

Every platform tool has its own internal RBAC model. Without a provisioning layer, each tool diverges: admins create users manually in GitLab, differently in NetBox, differently in Grafana. Authentik becomes SSO-only while tools become independent sources of truth for access. This is the problem this ADR solves.

---

## Decision

### Core principles

1. **Authentik is the platform IAM source of truth** — users, groups, and access policy live in Authentik. Tools are downstream consumers, never sources.
2. **Vault is the credentials source of truth** — all secrets (API tokens, SMTP credentials, OIDC client secrets) stored in Vault, never in git or tool config.
3. **Ansible is the provisioning engine** — one generic `identity-sync` role, per-tool adapters.
4. **Manual changes in tool UIs are break-glass only** — reconciliation cron corrects drift within 4h.
5. **Webhook is trigger only** — Ansible re-reads Authentik API for current truth before applying changes. Never trust webhook payload as full state.

---

## Architecture

```
                    ┌───────────────────────────┐
                    │         Authentik         │
                    │  source of truth for IAM  │
                    │ users / groups / policies │
                    └─────────────┬─────────────┘
                                  │
                 ┌────────────────┴────────────────┐
                 │                                 │
                 ▼                                 ▼
      ┌────────────────────┐            ┌────────────────────┐
      │   OIDC / SSO flow  │            │  Event / Webhook   │
      │   login only       │            │ provisioning trigger│
      └─────────┬──────────┘            └─────────┬──────────┘
                │                                 │
                ▼                                 ▼
      ┌────────────────────┐            ┌────────────────────┐
      │  Platform tool     │            │   GitLab CI        │
      │  authenticates     │            │  identity-sync     │
      │  user via Authentik│            │  pipeline          │
      └─────────┬──────────┘            └─────────┬──────────┘
                │                                 │
                └──────────────┬──────────────────┘
                               ▼
                    ┌───────────────────────────┐
                    │  Tool API (GitLab / NetBox │
                    │  / Vault / Grafana / ...)  │
                    │  create / block / update   │
                    │  users, groups, roles      │
                    └───────────────────────────┘
```

---

## When provisioning sync starts per tool

Identity sync does not exist before GitLab. It starts when GitLab is installed and active.
Each tool's adapter is activated when that tool is deployed. No adapter = no sync for that tool yet.

| Tool | Identity sync | Notes |
|---|---|---|
| OPNsense | None | Network appliance — local admin only (break-glass) |
| step-ca | None | PKI service — local admin only (break-glass) |
| Authentik | N/A | Source — not a sync target |
| Vault | Group → policy sync | Authentik groups → Vault policies via adapter |
| GitLab | Full user/group sync | First adapter. Users, groups, roles via API |
| NetBox | Full user/group sync | RBAC via NetBox permissions API |
| Grafana | Full user/group sync | Org + team role sync |
| Traefik | None | Routing only, no user model |
| Every subsequent tool | Add adapter | Same framework, new adapter vars file |

---

## Ansible framework

```
infra-ansible/
  roles/
    identity-sync/                ← generic framework (adapter pattern)
      tasks/
        main.yml                  ← orchestrates: read desired, read actual, diff, apply
        read-authentik.yml        ← Authentik API read (common to all adapters)
        diff.yml                  ← compute delta between desired and actual state
        alert.yml                 ← Discord alert on drift or error
      adapters/
        gitlab.yml                ← GitLab API calls
        netbox.yml                ← NetBox API calls
        vault.yml                 ← Vault policy sync
        grafana.yml               ← Grafana org/role sync
        msteams.yml               ← MS Teams via Graph API
        discord.yml               ← Discord role assignment
        exchange.yml              ← M365 mailboxes/groups via Graph API
        mailcow.yml               ← Mailcow mailboxes/aliases via REST API
      vars/
        gitlab-mapping.yml        ← Authentik group → GitLab role mapping
        netbox-mapping.yml
        vault-mapping.yml
        grafana-mapping.yml
        ...

  playbooks/
    identity-sync/
      gitlab.yml                  ← runs identity-sync role with gitlab adapter
      netbox.yml
      vault.yml
      ...
```

**Framework responsibilities:**
- Read desired state from Authentik API
- Read actual state from target tool API
- Compute diff
- Alert on unexpected drift (Discord notification)
- Log every run for audit
- Dry-run mode mandatory (`--check` flag)

**Adapter responsibilities:**
- Target tool API specifics only
- No Authentik-reading logic (handled by framework)

---

## Execution model

**Trigger:** Authentik event → webhook → GitLab CI pipeline → identity-sync playbook

**Steady-state executor:** GitLab CI (activated when GitLab is deployed)

**Reconciliation cron:** GitLab CI scheduled pipeline, every 4 hours — mandatory regardless of webhook health

**Why cron is mandatory even with webhooks:**

| Scenario | Webhook catches it? | Cron catches it? |
|---|---|---|
| Normal user change | ✅ | ✅ |
| Webhook missed or failed | ❌ | ✅ |
| Break-glass manual change in tool UI | ❌ | ✅ |
| Authentik DB restored from backup | ❌ | ✅ |
| Tool-side API bug introduced drift | ❌ | ✅ |

**Reconciliation behaviour on drift:**
1. User in tool but not in Authentik → block immediately + alert
2. User has higher role than mapping allows → downgrade + alert
3. User missing from group → add silently
4. Clean → log OK, no alert

Correct silently, alert always. Never delete automatically.

---

## Webhook security format

All Authentik → GitLab CI webhooks must conform to this format:

```
POST /sync/identity
Content-Type: application/json
X-Signature: sha256=<hmac-sha256-of-body-with-shared-secret>
X-Event-Type: user.created | user.modified | user.deleted | group.modified
X-Tool: gitlab | netbox | vault | all
X-Source: authentik
X-Timestamp: <unix-epoch-seconds>

{ event payload }
```

**Security requirements:**
1. HMAC-SHA256 signature on every webhook — shared secret in Vault at `secret/{env}/authentik/webhook-secret`
2. Receiver validates signature before processing anything
3. Replay protection: reject if `X-Timestamp` is > 5 minutes old
4. TLS only — no plain HTTP webhooks
5. Payload is trigger only — receiver re-reads Authentik API before applying changes

---

## Authentik group naming convention

_(Canonical definition is in ADR-0010 §9. Reproduced here for operational reference.)_

| Prefix | Meaning | Example |
|---|---|---|
| `tool-` | Access to a specific platform tool | `tool-gitlab-developers`, `tool-netbox-admins` |
| `svc-` | Service account groups | `svc-ansible-sync`, `svc-vault-agent` |
| `prj-` | Project-scoped groups | `prj-client-xyz-developers` |
| `org-` | Org-wide platform roles | `org-admins`, `org-readonly` |

Full pattern: `{prefix}{tool}-{role}` for tool groups, `{prefix}{project}-{role}` for project groups.
All lowercase, hyphen-separated. No uppercase, no underscores.

Group names describe **intent**, not the target platform. One group name, multiple platform outcomes — the mapping file decides which platform gets what.

---

## Group → role mapping (example — GitLab)

Stored in `roles/identity-sync/vars/gitlab-mapping.yml`. Never hardcoded in Authentik or GitLab.

```yaml
identity_sync:
  gitlab:
    enabled: true
    default_state: blocked
    groups:
      org-admins:
        target_group: by-systems
        access_level: owner
      tool-gitlab-maintainers:
        target_group: by-systems/platform
        access_level: maintainer
      tool-gitlab-developers:
        target_group: by-systems/platform
        access_level: developer
      tool-gitlab-reporters:
        target_group: by-systems/platform
        access_level: reporter
      prj-client-xyz-developers:
        target_project: by-systems/projects/client-xyz
        access_level: developer
```

Project-level access (contractors) is explicit per project in the vars file, not inferred.

---

## User removal policy

| State | Action | Tool state |
|---|---|---|
| User disabled in Authentik | Remove all group memberships + block in tool | `blocked` |
| User deleted in Authentik | Same as disabled | `blocked` (never auto-deleted) |
| Manual delete in tool (break-glass) | Cron detects drift, re-applies state | Corrected within 4h |

**Never hard-delete automatically.** Preserves: commit history, audit trail, merge request authorship, compliance evidence.
Hard deletion = explicit manual action by admin after audit confirmation.

Same policy applies to all tools where user history matters.

---

## Tool bootstrap & admin account lifecycle

Every tool deployed on this platform follows a 3-step bootstrap pattern:

| Step | Action | How |
|---|---|---|
| **1. Bootstrap** | Deploy creates default admin | Credentials written to Vault immediately at deploy time |
| **2. Rotate** | Change default credential | Ansible rotates post-deploy, stores new value in Vault |
| **3. Federate** | Normal admin work via SSO | Authentik-federated admin account for day-to-day ops |

Break-glass = local admin only. Vault path: `secret/{env}/{tool}/break-glass`. Wazuh alert on any break-glass use.

**Per-tool admin account map:**

| Tool | Bootstrap account | Post-deploy action | Break-glass Vault path |
|---|---|---|---|
| Authentik | `akadmin` | Rotate + store in Vault | `secret/{env}/authentik/break-glass` |
| GitLab | `root` | Rotate + store in Vault | `secret/{env}/gitlab/break-glass` |
| Vault | Root token | Revoke after init — unseal keys in cold storage | `secret/{env}/vault/unseal-keys` |
| NetBox | Django superuser | Rotate + store in Vault | `secret/{env}/netbox/break-glass` |
| Grafana | `admin` | Rotate + store in Vault | `secret/{env}/grafana/break-glass` |
| Mailcow | `admin` (superadmin) | Rotate + store in Vault | `secret/{env}/mailcow/break-glass` |
| OPNsense | `root` | Rotate + store in Vault | `secret/{env}/opnsense/break-glass` |

**Vault root token special case:** Root token is not kept active. After init: configure Vault (auth, policies, engines) → revoke root token → store unseal keys in encrypted cold storage.

---

## Per-tool identity documentation standard

Every tool's `tools/{tool}/docs/identity.md` must contain:
1. Reference to this ADR (not a copy of the rules)
2. Authentik application name, provider type (OIDC/SAML), scopes, redirect URIs
3. Vault secret paths for this tool's credentials (format per ADR-0016)
4. Ansible adapter name + vars file location
5. Group → role mapping for this tool (link to vars file, not inline copy)
6. Removal policy specifics (tool-specific notes if any)

---

## Consequences

- Tools are never manually configured for user access in normal operation
- GitLab UI user management is break-glass only — cron corrects within 4h
- All secrets via Vault — no credentials in git or markdown
- Ansible vars files are the single source of group→role mapping per tool
- Adding a new tool = write one adapter + one vars file, nothing else changes
- Dry-run mode is mandatory before any production sync run

---

## CISO mapping

> Applies only to controls directly relevant to this ADR's scope.

### ISO/IEC 27001:2022

| Control | Title | Status | Notes |
|---|---|---|---|
| A.5.15 | Access control | ✓ Covered | Group-based RBAC enforced via Authentik → Ansible sync |
| A.5.16 | Identity management | ✓ Covered | User lifecycle managed in Authentik, propagated to all tools |
| A.5.18 | Access rights | ✓ Covered | Mapping files define least-privilege role per group per tool |
| A.8.2 | Privileged access rights | ✓ Covered | Break-glass accounts in Vault; Wazuh alert on use |
| A.9.2.1 | User registration | ✓ Covered | Tool bootstrap standard defines admin lifecycle explicitly |
| A.9.2.3 | Privileged access | ✓ Covered | Break-glass policy: block + alert on any break-glass use |
| A.9.2.5 | Access rights review | ✓ Covered | Reconciliation cron replaces manual quarterly review for platform tools |
| A.9.2.6 | Removal of access rights | ✓ Covered | Block-not-delete policy, group removal on user disable |
| A.8.24 | Use of cryptography | ✓ Covered | Webhook HMAC-SHA256 signature on all provisioning events |

### NIS2 (Directive 2022/2555)

| Article | Requirement | Status | Notes |
|---|---|---|---|
| Art. 21(2)(a) | Risk management | ✓ Covered | Automated reconciliation reduces access drift risk |
| Art. 21(2)(i) | Human resources security | ✓ Covered | User removal: block immediately, all group memberships removed, audit trail preserved |

---

## References

- ADR-0004: SSO / authentication architecture (this ADR extends it with provisioning)
- ADR-0010: Naming convention §9 (Authentik group naming prefixes)
- ADR-0016: Vault KV path convention
- Brainstorm: `brainstorming/2026-04-02-authentik-gitlab-identity-sync-brainstorm.md`
