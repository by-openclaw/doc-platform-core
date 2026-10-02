# identity/0002 — Provisioning & Sync

**Status:** Draft
**Date:** 2026-04-12 (supersedes flat ADR-0024, 2026-04-02); amended 2026-10-02 — break-glass = the bootstrap admin credential at `secret/{env}/{tool}/admin`, alert on any read by a non-deploy identity; the 4-hour reconciliation runs as a GitLab scheduled pipeline on the platform runner
**Scope:** How identity propagates from Authentik to platform tools.
**Related:** `identity/0001-authentication`, `naming/0001-naming-convention` §Authentik groups, `security/0001-secret-storage`

---

## Context

Authentik tells a tool who a user is. It does **not** create the user inside that tool, assign groups, or apply roles. Without a provisioning layer, each tool diverges. This ADR defines the sync framework.

## Decision

### Core principles

1. **Authentik = IAM source of truth** — users, groups, policies live in Authentik. Tools are downstream consumers.
2. **Vault = credentials source of truth** — all API tokens, client secrets in Vault.
3. **Ansible = provisioning engine** — one generic `identity-sync` role, per-tool adapters.
4. **Manual changes in tool UIs = break-glass only** — reconciliation corrects drift within 4h. The reconciliation is a **GitLab scheduled pipeline** (every 4h) on the platform runner, running the identity playbook with the deploy AppRole; on-demand runs from an operator workstation are for development and incident response, never the only path.
5. **Webhook = trigger only** — Ansible re-reads Authentik API before applying. Never trust webhook payload as state.

## Architecture

```
Authentik (source of truth)
    │
    ├── OIDC/SSO ─────► platform tool (login)
    │
    └── event/webhook ──► GitLab CI ──► identity-sync playbook ──► tool API
                                              ▲
                        4h reconciliation cron┘
```

## Sync scope per tool

| Tool | Sync | Notes |
|---|---|---|
| Authentik | N/A | Source |
| Vault | Group → policy | Authentik groups → Vault policies |
| GitLab | Full user/group | First adapter |
| NetBox | Full user/group | RBAC via permissions API |
| Grafana | Full user/group | Org + team role sync |
| OPNsense / step-ca | None | Local admin only (break-glass) |
| Traefik | None | No user model |

Each tool's adapter activates when that tool is deployed. No adapter = no sync yet.

## Ansible framework

```
infra-ansible/roles/identity-sync/
├── tasks/
│   ├── main.yml              # read desired → read actual → diff → apply
│   ├── read-authentik.yml    # common to all adapters
│   ├── diff.yml
│   └── alert.yml             # Discord on drift/error
├── adapters/                 # one per tool: gitlab, netbox, vault, grafana, ...
└── vars/                     # Authentik group → tool role mappings
```

**Framework:** reads Authentik, diffs, alerts, logs, supports `--check` (dry-run mandatory).
**Adapters:** tool-specific API calls only. No Authentik logic.

## Execution model

- **Trigger:** Authentik event → webhook → GitLab CI → identity-sync playbook
- **Reconciliation cron:** every 4h — mandatory regardless of webhook health
- **Drift actions:**
  1. User in tool but not in Authentik → block + alert
  2. User has higher role than mapping allows → downgrade + alert
  3. User missing from group → add silently
  4. Clean → log OK

**Never delete automatically.** Correct silently, alert always.

## Webhook security

All Authentik → GitLab CI webhooks:

- HMAC-SHA256 signature (secret in Vault `secret/{env}/authentik/webhook-secret`)
- Replay protection: reject if `X-Timestamp` > 5 min old
- TLS only
- Payload is trigger — receiver always re-reads Authentik API

Headers: `X-Signature`, `X-Event-Type`, `X-Tool`, `X-Source`, `X-Timestamp`.

## Group → role mapping

Stored per tool in `roles/identity-sync/vars/{tool}-mapping.yml`. Never hardcoded.

```yaml
identity_sync:
  gitlab:
    enabled: true
    default_state: blocked
    groups:
      org-admins:            { target_group: by-systems, access_level: owner }
      tool-gitlab-developers: { target_group: by-systems/platform, access_level: developer }
      prj-client-xyz-devs:   { target_project: by-systems/projects/client-xyz, access_level: developer }
```

Authentik group naming convention (prefixes `tool-`, `svc-`, `prj-`, `org-`): see `naming/0001-naming-convention` §Authentik groups.

## User removal policy

| State | Action | Tool state |
|---|---|---|
| Disabled in Authentik | Remove groups + block in tool | `blocked` |
| Deleted in Authentik | Same as disabled | `blocked` (never auto-deleted) |
| Manual delete in tool | Cron detects drift, re-applies | Corrected ≤4h |

**Never hard-delete automatically.** Preserves commit history, audit trail, compliance evidence.

## Tool bootstrap standard

Every tool follows 3 steps:

| Step | Action |
|---|---|
| 1. Bootstrap | Deploy creates default admin → credentials written to Vault |
| 2. Rotate | Ansible rotates post-deploy |
| 3. Federate | Day-to-day via Authentik SSO |

Break-glass credential = the bootstrap admin of step 1, at `secret/{env}/{tool}/admin` (one path per tool; no separate copy — a second path would be a second secret to rotate and audit). **Alert on any use:** Vault's audit log is shipped to the log platform; a read of any `secret/{env}/*/admin` path by an identity other than the deploy AppRole raises an alert (Wazuh/Alertmanager) to the on-call channel.

**Vault special case:** root token revoked after init; unseal keys in encrypted cold storage.

## Per-tool identity doc requirement

Every tool's `tools/{tool}/docs/identity.md` must list:
1. Reference to this ADR
2. Authentik app name, provider type, scopes, redirect URIs
3. Vault paths for this tool's credentials
4. Ansible adapter + vars file location
5. Group → role mapping link (not inline copy)
6. Tool-specific removal notes

## Consequences

- Tools are never manually configured in normal operation
- GitLab UI user mgmt = break-glass only
- Adding a new tool = one adapter + one vars file
- Dry-run mandatory before any production sync

## CISO mapping

| Framework | Controls covered |
|---|---|
| ISO 27001:2022 | A.5.15, A.5.16, A.5.18, A.8.2, A.9.2.1, A.9.2.3, A.9.2.5, A.9.2.6, A.8.24 |
| NIS2 | Art. 21(2)(a), 21(2)(i) |
| GDPR | Art. 32 |
