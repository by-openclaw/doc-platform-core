# ADR-0025: Platform Notification & Collaboration Integration

**Date:** 2026-04-02
**Status:** Accepted
**Deciders:** yboujraf, Rune
**Extends:** ADR-0024 (identity sync framework), ADR-0016 (Vault paths)

---

## Context

Platform tools (GitLab, NetBox, Grafana, Vault, etc.) generate events: pipeline failures, merge requests, user logins, alerts, ticket updates. Without a standard, each tool sends notifications wherever its default config points — different channels, different formats, no routing logic, no consistency.

The same identity-sync framework (ADR-0024) that manages user/group provisioning must also manage collaboration platform integration: routing events to the correct Teams channel, Discord channel, or email address. The notification layer and the provisioning layer use the same Ansible adapter pattern.

---

## Decision

### Core principles

1. **One-way only.** Tool → channel. No interactive elements, no buttons, no reply handling in this phase. Two-way (slash commands, approvals via chat) is deferred to Phase 2.
2. **Deep link mandatory.** Every notification must include a direct link to the source object.
3. **Correct channel per event type.** No broadcasting to `#general`. Each event type has a defined destination.
4. **Template standardized.** Same format across all tools and platforms.
5. **Slack: out of scope.** SSO requires a paid plan. Deferred until paid plan is confirmed.

---

## Supported platforms

| Platform | Protocol | Scope | Notes |
|---|---|---|---|
| MS Teams | Graph API | User/group provisioning + event notifications | Requires Azure app registration |
| Discord | Discord API | Role assignment + event notifications | User must join server manually first |
| Email (M365) | Graph API | Distribution group management + notifications | Licensed users and shared mailboxes |
| Email (Mailcow) | REST API | Platform tool mailboxes + internal routing | PoC/internal tier |
| Slack | — | **Out of scope** | Paid plan required for SSO/SCIM. Revisit when plan confirmed |

---

## Notification template standard

All platform notifications — regardless of source tool or destination platform — use this format:

```
🔔 [{Tool}] {Event type}
{One-line summary}
Who:  {actor username or "system"}
What: {object name / title}
Link: {https://tool.{env}.{domain}/path/to/object}
```

**Examples:**

```
🔔 [GitLab] Merge Request opened
Add Traefik TLS config for Vault
Who:  yboujraf
What: MR !42 → main
Link: https://gitlab.poc.example.com/by-systems/infra-ansible/-/merge_requests/42

🔔 [GitLab] Pipeline failed
Stage: test | Job: lint
Who:  CI system
What: infra-ansible / branch: feature/vault-tls
Link: https://gitlab.poc.example.com/by-systems/infra-ansible/-/pipelines/117

🔔 [Grafana] Alert fired
CPU > 90% for 5 min
Who:  system
What: vm-gitlab-poc-01 | alert: high-cpu
Link: https://grafana.poc.example.com/alerting/alert/42

🔔 [Vault] Break-glass access used
Who:  adm_yboujraf
What: secret/poc/gitlab/break-glass (read)
Link: https://vault.poc.example.com/ui/vault/secrets/poc/gitlab
```

**Rules:**
- `{Tool}` is always capitalized and matches the tool's canonical name
- `Who:` is never empty — use `"system"` for automated/CI events
- `Link:` is always a direct link to the specific object — never the tool homepage
- One-line summary: max 80 characters, no markdown
- No emoji beyond the leading 🔔 (platform renders differently)

---

## Event routing matrix

Per tool, define which events route to which channel. This table is the platform default — per-tool docs may refine.

| Source tool | Event type | Teams channel | Discord channel |
|---|---|---|---|
| GitLab | MR opened / updated | #platform-engineering | #platform-dev |
| GitLab | Pipeline failed | #platform-alerts | #platform-alerts |
| GitLab | Pipeline passed | — (no noise) | — |
| GitLab | Issue created | #platform-engineering | #platform-dev |
| GitLab | Release published | #platform-releases | #releases |
| Grafana | Alert fired | #platform-alerts | #platform-alerts |
| Grafana | Alert resolved | #platform-alerts | #platform-alerts |
| Vault | Break-glass access | #security-alerts | #security-alerts |
| Vault | Secret rotation failed | #platform-alerts | #platform-alerts |
| Authentik | Login failed (>3) | #security-alerts | #security-alerts |
| Authentik | New user created | #platform-ops | — |
| NetBox | Config change | #platform-ops | — |
| Wazuh | Critical alert | #security-alerts | #security-alerts |

Channels not listed for a platform = that event is not routed there. Do not default to `#general`.

---

## Collaboration platform object model

| Concept | MS Teams | Discord |
|---|---|---|
| Container | Team | Server / Guild |
| Sub-container | Channel | Channel |
| Role in container | Owner / Member / Guest | Role |
| Notification target | Channel / User | Channel / Role / User |
| Private sub-space | Private Channel | Private Channel |

---

## MS Teams — provisioning model

**Protocol:** Microsoft Graph API
**App registration:** Azure AD app with scopes `Group.ReadWrite.All`, `TeamMember.ReadWrite.All`, `ChannelMember.ReadWrite.All`
**Credentials:** Vault at `secret/{env}/msteams/graph-api`

**What Ansible provisions:**

| Object | Action |
|---|---|
| Team | Create on new org/project group |
| Channel within Team | Create per function/scope |
| User → Team | Add / remove with role (Owner / Member) |
| User → private Channel | Add / remove |
| User `absent` | Remove from all Teams |

**Mapping example:**
```yaml
identity_sync:
  msteams:
    enabled: true
    groups:
      org-admins:
        team: BY-SYSTEMS
        role: owner
      tool-platform-devs:
        team: BY-SYSTEMS
        channels:
          - Platform Engineering
        role: member
      prj-client-xyz-developers:
        team: Projects
        channels:
          - client-xyz
        role: member
```

---

## Discord — provisioning model

**Protocol:** Discord API (bot token)
**Bot permissions:** `Manage Roles`, `Manage Members`
**Credentials:** Vault at `secret/{env}/discord/bot-token`

**Constraints:**
- Discord server (Guild) is created manually — cannot be automated via API
- Users must join the server via invite link before role assignment is possible
- After join: role assignment is fully API-driven

**What Ansible provisions:**

| Object | Action |
|---|---|
| Role within server | Create per Authentik group (if not exists) |
| User → Role | Assign / revoke |
| User `absent` | Revoke all roles (cannot programmatically remove user from server without kick) |

**Channel access:** controlled by Role permissions — not per-user. Assign correct role, channel access follows.

**Mapping example:**
```yaml
identity_sync:
  discord:
    enabled: true
    groups:
      org-admins:
        roles:
          - admin
          - staff
      tool-platform-devs:
        roles:
          - platform-team
      prj-client-xyz-developers:
        roles:
          - contractors
```

---

## Email provisioning — M365 (Exchange via Graph API)

**Protocol:** Microsoft Graph API
**Credentials:** Vault at `secret/{env}/msteams/graph-api` (same app registration as Teams)

**What Ansible provisions:**

| Object | Action |
|---|---|
| Distribution Group | Create / delete |
| Mail-enabled Security Group | Create / delete |
| User → Group | Add / remove |
| Shared mailbox access | Grant / revoke `FullAccess` |
| User `absent` | Remove from all groups, disable mailbox |

---

## Email provisioning — Mailcow (REST API)

**Protocol:** Mailcow REST API (Bearer token)
**Credentials:** Vault at `secret/{env}/mailcow/api-key`

**What Ansible provisions:**

| Object | Action |
|---|---|
| Domain | Create (per deployment) |
| Mailbox | Create / delete (tool mailboxes, not user mailboxes) |
| Alias (including catchall) | Create / delete |
| DKIM | Generate and configure |
| Shared folder ACL | Grant / revoke |

Tool mailbox naming: `{tool}@{domain}` (e.g. `gitlab@by-systems.be`, `netbox@by-systems.be`).

---

## Ansible framework additions

Notification adapters added to the `identity-sync` role from ADR-0024:

```
roles/identity-sync/adapters/
  gitlab.yml
  netbox.yml
  vault.yml
  grafana.yml
  msteams.yml       ← Teams provisioning + notification routing
  discord.yml       ← Discord role provisioning + notification routing
  exchange.yml      ← M365 mailboxes/groups
  mailcow.yml       ← Mailcow mailboxes/aliases

playbooks/identity-sync/
  msteams.yml
  discord.yml
  exchange.yml
  mailcow.yml
```

---

## Two-way interactive notifications — deferred

Slash commands, approval buttons, and chat-driven actions (e.g. `/approve MR !42` in Discord, clicking an approval button in Teams) are explicitly deferred to Phase 2.

Phase 1 delivers: tool → channel (one-way, templated, routed).

---

## Per-tool notification documentation standard

Every tool's `tools/{tool}/docs/notifications.md` must contain:
1. Reference to this ADR (not a copy of the rules)
2. Which events this tool fires
3. Which Teams/Discord channel receives each event type (reference routing matrix above, note deviations)
4. Notification template used (reference this ADR's template format, note tool-specific fields)
5. SMTP config notes if tool sends its own email (link to `tools/{tool}/docs/setup.md`)

---

## Consequences

- Notification format is consistent across all tools — no per-tool formatting decisions
- Channel routing is explicit — no undirected `#general` noise
- Slack omission is deliberate and documented — revisit when paid plan is confirmed
- Two-way interactivity deferred — Phase 1 is send-only
- Adding a new notification source = add mapping to routing matrix + update tool notification doc
- All platform credentials (Graph API, bot tokens, Mailcow API) managed via Vault

---

## CISO mapping

> Applies only to controls directly relevant to this ADR's scope.

### ISO/IEC 27001:2022

| Control | Title | Status | Notes |
|---|---|---|---|
| A.8.15 | Logging | ✓ Covered | Security alerts (break-glass, failed logins) routed to dedicated security channel |
| A.5.16 | Identity management | ✓ Covered | Teams/Discord group membership follows Authentik source of truth |
| A.6.8 | Reporting information security events | ✓ Covered | Wazuh + Vault + Authentik alerts routed to `#security-alerts` channel |

### NIS2 (Directive 2022/2555)

| Article | Requirement | Status | Notes |
|---|---|---|---|
| Art. 21(2)(b) | Incident handling | ✓ Covered | Security events routed to `#security-alerts` with deep link for immediate response |

---

## References

- ADR-0024: Identity Provisioning & Sync Standard (adapter pattern this ADR extends)
- ADR-0026: Email Infrastructure Standard (inbound routing, Mailcow + M365 dual stack)
- ADR-0016: Vault KV path convention
- Brainstorm: `brainstorming/2026-04-02-authentik-gitlab-identity-sync-brainstorm.md`
