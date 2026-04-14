# services/0005 — Notification & Collaboration Integration

**Status:** Draft
**Date:** 2026-04-14 (supersedes flat ADR-0025, 2026-04-02)
**Scope:** Event notification routing from platform tools to collaboration channels (Teams, Discord, email). Standard message template, per-tool event routing matrix, credential model. Does not define identity-sync adapter mechanics (see `identity/0002-provisioning`) or mail infrastructure (see `services/0002-email-infrastructure`).
**Related:** `identity/0002-provisioning`, `services/0002-email-infrastructure`, `security/0001-secret-storage`, `infra/0007-monitoring`

---

## Context

Platform tools (GitLab, NetBox, Grafana, Vault, Authentik, Wazuh, …) generate events: pipeline failures, merge requests, user logins, alerts, ticket updates. Without a standard, every tool sends notifications wherever its default config points — different channels, different formats, no routing logic, no consistency, no deep links.

Notification routing uses the **same Ansible adapter pattern as identity provisioning** (`identity/0002-provisioning`). Rather than re-inventing a notification framework, the `identity-sync` role gets additional adapters — one per collaboration platform — that handle both user/group provisioning **and** event routing. One framework, two responsibilities.

## Decision

### 1. Core principles

1. **One-way only.** Tool → channel. No buttons, no slash commands, no reply handling. Two-way interactive notifications (slash commands, approval buttons, chat-driven actions) are explicitly deferred.
2. **Deep link mandatory.** Every notification includes a direct link to the source object — never the tool homepage.
3. **Correct channel per event type.** No broadcasting to `#general`. Each event type has a defined destination in the §Event routing matrix.
4. **Template standardized.** Same format across every tool and every platform — no per-tool formatting choices.
5. **Slack: out of scope.** Slack SSO / SCIM require a paid plan. Deferred until a paid plan is confirmed — tracked in `security/0002-compliance-mapping §Known gaps`.

### 2. Notification template

All platform notifications use this format — regardless of source tool or destination platform:

```
🔔 [{Tool}] {Event type}
{One-line summary, ≤ 80 chars, no markdown}
Who:  {actor username or "system"}
What: {object name or title}
Link: {https://tool.{env}.{domain}/path/to/object}
```

**Rules:**
- `{Tool}` is capitalized and matches the tool's canonical name (GitLab, NetBox, Grafana, …)
- `Who:` is **never empty** — use `"system"` for automated / CI events
- `Link:` is always a direct link to the specific object — never the tool homepage or a search page
- One-line summary: **max 80 characters**, no markdown
- No emoji beyond the leading `🔔` — downstream platforms render custom emoji inconsistently

**Examples:**

```
🔔 [GitLab] Merge Request opened
Add Traefik TLS config for Vault
Who:  yboujraf
What: MR !42 → main
Link: https://gitlab.by-research.be/by-systems/infra-ansible/-/merge_requests/42

🔔 [Grafana] Alert fired
CPU > 90% for 5 min
Who:  system
What: vm-glab-01 | alert: high-cpu
Link: https://grafana.by-research.be/alerting/alert/42

🔔 [Vault] Break-glass access used
Who:  adm_yboujraf
What: secret/dev/gitlab/break-glass (read)
Link: https://vault.by-research.be/ui/vault/secrets/dev/gitlab
```

### 3. Event routing matrix

This table is the **platform default**. Per-tool `docs/notifications.md` files may refine it, but cannot add routes that default to `#general`.

| Source tool | Event | Teams channel | Discord channel |
|---|---|---|---|
| GitLab | MR opened / updated | `#platform-engineering` | `#platform-dev` |
| GitLab | Pipeline failed | `#platform-alerts` | `#platform-alerts` |
| GitLab | Pipeline passed | — (no noise) | — |
| GitLab | Issue created | `#platform-engineering` | `#platform-dev` |
| GitLab | Release published | `#platform-releases` | `#releases` |
| Grafana | Alert fired | `#platform-alerts` | `#platform-alerts` |
| Grafana | Alert resolved | `#platform-alerts` | `#platform-alerts` |
| Vault | Break-glass access | `#security-alerts` | `#security-alerts` |
| Vault | Secret rotation failed | `#platform-alerts` | `#platform-alerts` |
| Authentik | Login failed (> 3 in 5 min) | `#security-alerts` | `#security-alerts` |
| Authentik | New user created | `#platform-ops` | — |
| NetBox | Config change | `#platform-ops` | — |
| Wazuh | Critical alert | `#security-alerts` | `#security-alerts` |

**Rules:**
- Channels **not listed** for a platform = that event is not routed there. Do **not** default to `#general`.
- Adding a new event source = add a row to this matrix via ADR amendment.
- Pipeline **passed** is deliberately not routed — success notifications create alert fatigue.

### 4. Supported collaboration platforms

| Platform | Protocol | Purpose | Credential location |
|---|---|---|---|
| **MS Teams** | Microsoft Graph API | User / group provisioning + event notifications | `secret/{env}/msteams/graph-api` |
| **Discord** | Discord bot API | Role assignment + event notifications | `secret/{env}/discord/bot-token` |
| **M365 Email** | Microsoft Graph API | Distribution group management + notifications | `secret/{env}/msteams/graph-api` (same Azure app registration as Teams) |
| **Mailcow Email** | Mailcow REST API | Platform tool mailboxes + internal routing | `secret/{env}/mailcow/api-key` |
| **Slack** | — | **Out of scope** — revisit when paid plan is confirmed | — |

All credential paths follow `security/0001-secret-storage §KV v2 path convention`.

### 5. Adapter mechanics — delegated to identity-sync framework

Teams / Discord / Exchange / Mailcow adapters are implemented as **adapters to the `identity-sync` Ansible role** defined in `identity/0002-provisioning §Ansible framework`. They are **not** a separate framework.

Each adapter handles both responsibilities under one role:

- **Provisioning:** create teams / channels / roles / mailboxes, add users to them, handle the `absent` state (remove user from all teams / revoke all roles / disable mailbox)
- **Event routing:** receive events from source tools (webhook or polling), apply the §3 routing matrix, render the §2 template, post to the correct channel

Adapter-specific rules, protocols, and Azure / Discord / Mailcow API shapes **belong in `identity/0002-provisioning`**, not here. This ADR owns the **routing decisions** (template, matrix, credential paths); mechanics are delegated.

### 6. Platform-specific constraints

These constraints are architecturally significant enough to call out in this ADR:

**MS Teams:**
- Azure app registration required with scopes `Group.ReadWrite.All`, `TeamMember.ReadWrite.All`, `ChannelMember.ReadWrite.All`
- Teams are the top-level container; Channels are sub-containers
- Notification target: channel or user

**Discord:**
- **Server (Guild) is created manually** — cannot be automated via API. One-time manual setup per deployment.
- **Users must join the server via invite link before API role assignment is possible.** After join, role assignment is fully API-driven.
- **User `absent` cannot remove a user from the server** without kicking them — role revocation only. Kick is a privileged action reserved for humans.
- Channel access is controlled by role permissions — never per-user. Assign the correct role, channel access follows.

**M365 Email (via Graph API):**
- Same Azure app registration as Teams (shared credential)
- Distribution groups, mail-enabled security groups, shared mailbox `FullAccess` grants

**Mailcow Email (via REST API):**
- See `services/0002-email-infrastructure` for the mail flow architecture
- Domains, mailboxes (tool mailboxes only — never user mailboxes), aliases (including catch-all), DKIM, shared folder ACL

### 7. Per-tool notification documentation standard

Every tool's `platform-setup/tools/{tool}/docs/notifications.md` must contain:

1. Reference to this ADR (not a copy of the template or matrix)
2. Which events the tool fires
3. Which channel receives each event type — reference §3 routing matrix, document deviations explicitly
4. Notification template used — reference §2, document tool-specific field overrides
5. SMTP config notes (if the tool sends its own email) — link to `platform-setup/tools/{tool}/docs/setup.md`

Operational procedures (how to add a new mailbox, how to rotate a bot token, how to recover a lost Azure app secret) are runbook content and live in the tool's `runbooks/` directory, not in this ADR.

### 8. Deferred: two-way interactive notifications

Slash commands (`/approve MR !42`), approval buttons, and chat-driven actions are **explicitly deferred**. The current phase delivers one-way, templated, routed notifications only.

Revisit when:
- A workflow explicitly requires chat-driven approvals (e.g. deploy gate confirmation)
- The operational benefit of chat-based actions exceeds the security complexity of authenticating them back to the source tool

## Consequences

- **Uniform notification format** across every tool and every platform — no per-tool formatting decisions, auditor-friendly
- **Channel routing is explicit** — no undirected `#general` noise, every event has a defined destination
- **Slack absence is documented** — not an oversight, deliberately deferred until paid plan
- **Two-way interactivity deferred** — current scope is send-only, no approval-via-chat
- **Adapter mechanics live in one framework** (`identity-sync`, `identity/0002-provisioning`) — notification and provisioning share the same Ansible code
- **Credentials in Vault only** — Graph API tokens, Discord bot tokens, Mailcow API keys all at well-defined Vault paths per `security/0001-secret-storage`
- **Event routing matrix is versioned** — adding a new source, channel, or event type requires an ADR amendment, no ad-hoc rules

## Revision triggers

Revise this ADR when:
- A new collaboration platform is supported (e.g. Slack after paid plan, Zulip, Rocket.Chat)
- The notification template format changes (new mandatory field, new emoji rule)
- Two-way interactive notifications are enabled (removes §8 deferral)
- A new event source is added to the routing matrix
- The routing matrix needs per-env channels (currently one set of channels for all envs — may need split once `drp` is active)
- Azure app registration scopes change for Teams / M365

## CISO mapping

| Framework | Controls covered |
|---|---|
| ISO 27001:2022 | A.8.15 (logging — security alerts routed to dedicated security channel), A.5.16 (identity management — Teams/Discord group membership follows Authentik source of truth), A.6.8 (reporting information security events — Wazuh + Vault + Authentik alerts to `#security-alerts`) |
| NIS2 | Art. 21(2)(b) (incident handling — security events routed with deep link for immediate response) |
