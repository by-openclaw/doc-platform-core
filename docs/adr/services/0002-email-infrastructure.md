# services/0002 — Email Infrastructure

**Status:** Draft
**Date:** 2026-04-14 (supersedes flat ADR-0026, 2026-04-02)
**Scope:** Platform mail flow architecture — inbound routing, outbound relay, per-tool mailbox model, catch-all for virtual addresses. Does not define mailbox lifecycle procedures (add/remove/onboard/offboard) — those are runbook content.
**Related:** `identity/0002-provisioning`, `security/0001-secret-storage`, `security/0004-certificate-strategy`, `naming/0001-infra §9`, `infra/0002-platform-charter §Layer 5`

---

## Context

The platform needs a structured email infrastructure that:
- Receives inbound email for every platform address under `@{domain}`
- Routes to the correct tool mailbox, or falls back to catch-all
- Provides outbound SMTP for every tool
- Requires **zero per-tool M365 licenses** (each licensed mailbox costs money per user per month)

There are two deployment models, and the decision rule between them is which layer "owns" the public MX record.

## Decision

### Two deployment models

| Model | MX owner | Exchange required | When used |
|---|---|---|---|
| **Mailcow standalone** | Mailcow is the sole MX — public MX record points directly at the Mailcow host | ❌ No | Default. Simple installations, self-hosted orgs, customer deployments without M365 |
| **Mailcow + Exchange hybrid** | Exchange (M365) is the public MX; unknown recipients relay to Mailcow via two connectors | ✅ Yes | Orgs that already have an M365 tenant and want to keep licensed user mailboxes on Exchange while hosting tool mailboxes on Mailcow |

**The decision rule is per deployment, not per phase.** A deployment that never has M365 stays on Mailcow standalone forever. A deployment that already has M365 runs hybrid from day one. Hybrid is not a "future upgrade" of standalone.

### Core invariants (both models)

- **Every tool mailbox lives on Mailcow, never on Exchange.** Zero per-tool M365 licenses.
- **Catch-all (`catchall@{domain}`) is a real Mailcow mailbox.** It receives mail for virtual addresses (`sales@`, `accounting@`, `project-xyz@`) that exist only inside a tool, not in Exchange or Mailcow.
- **Subaddressing (`tool+token@{domain}`) is native.** Mailcow handles it without configuration; Exchange requires `AllowPlusAddressInRecipients` when in hybrid.
- **Outbound always flows through Mailcow's SMTP.** Tools send to `mcow:25`, Mailcow either delivers directly (standalone) or relays via Exchange (hybrid).
- **Credentials are in HashiCorp Vault** at `secret/{env}/mail/{toolname}` per `security/0001-secret-storage`. No passwords in tool config, no IMAP credentials on disk.
- **TLS everywhere** — IMAP over SSL, SMTP submission with STARTTLS, relay over TLS. No plaintext. Cert issuance per `security/0004-certificate-strategy`.

### Model 1 — Mailcow standalone

```
External sender → anyaddress@{domain}
       │
       ▼
┌─────────────────────────────────────────────┐
│  Mailcow (sole MX)                          │
│  Public MX record → Mailcow host            │
│                                             │
│  Known tool mailbox (gitlab@, grafana@)?    │
│  └── YES → deliver to mailbox               │
│            Tool reads via IMAP (Vault cred) │
│                                             │
│  Unknown (virtual address)?                 │
│  └── NO  → catchall@{domain}                │
│            Consumer reads via IMAP,         │
│            inspects To: header, routes      │
│                                             │
│  Outbound: Tool → Mailcow SMTP → direct     │
└─────────────────────────────────────────────┘
```

No Exchange, no relay dependency, no external SaaS in the mail path.

### Model 2 — Mailcow + Exchange hybrid

Used when the org already has an M365 tenant with licensed user mailboxes. Exchange stays as public MX; unknown recipients are routed to Mailcow.

```
External sender → anyaddress@{domain}
       │
       ▼
┌─────────────────────────────────────────────┐
│  M365 Exchange (public MX)                  │
│  Domain type: Internal Relay                │
│                                             │
│  Licensed Exchange user? → deliver          │
│  Otherwise → Connector 1: Exchange → Mailcow│
└──────────────────────┬──────────────────────┘
                       │ Connector 1 (inbound)
                       ▼
┌─────────────────────────────────────────────┐
│  Mailcow                                    │
│  Mode: Relay non-existing mailboxes only    │
│                                             │
│  Known tool mailbox? → deliver              │
│  Unknown? → catchall@{domain}               │
│                                             │
│  Outbound: Tool → Mailcow → Connector 2     │
└──────────────────────┬──────────────────────┘
                       │ Connector 2 (outbound)
                       ▼
┌─────────────────────────────────────────────┐
│  M365 Exchange (outbound relay)             │
│  Auth: TLS cert or static IP                │
│  Delivers to external recipient             │
│  From: tool@{domain}                        │
└─────────────────────────────────────────────┘
```

**Two connectors, not N Mail Flow Rules:**

| Connector | Direction | Purpose |
|---|---|---|
| Connector 1 | Exchange → Mailcow | Route unknown-to-Exchange recipients to Mailcow |
| Connector 2 | Mailcow → Exchange | Mailcow outbound relay via Exchange |

Connector 2 authenticates via **TLS certificate** (subject = accepted domain in Exchange) or a static Mailcow IP.

**Mailcow relay safety:** the `Relay non-existing mailboxes only` setting is non-negotiable. It prevents Mailcow from becoming an open relay — Mailcow accepts relay only for recipients it has no mailbox for, otherwise it delivers locally.

### Mailcow deployment

Mailcow is a **Linux VM** deployed through the **generic Linux VM pipeline** (`infra/0002-platform-charter §Layer 1` → cloud-init → `identity/0004-os-accounts §9` → `security/0003-hardening`). It is **not** an exception like OPNsense — no dedicated provisioning contract ADR needed.

**Key deployment constraints:**

- **Uses the official `mailcow/mailcow-dockerized` image stack** — not a custom build. Mailcow ships with its own internal MariaDB and Redis inside the docker-compose file.
- **Does NOT consume the shared PostgreSQL / Redis** from `services/0004-database-strategy`. Reconfiguring Mailcow to use external DB/Redis is off the supported path and makes every upgrade a custom integration problem.
- **Host platform is flexible:** Mailcow can run on-prem (`vm-mcow-01` on Proxmox per `naming/0001-infra §5`) or on a rented VPS (`cb-mcow-01` for Contabo). The ADR does not mandate a specific host — it mandates the deployment pattern.
- **DKIM / SPF / DMARC are configured per domain** via the Mailcow API by Ansible. See `naming/0001-infra §9` for DNS zone rules.

### Mailbox naming

Pattern: `{tool}@{domain}` for send-only / general outbound. `{tool}-incoming@{domain}` when the tool needs a dedicated inbound with `+token` subaddressing.

Mailbox creation is an Ansible task against the Mailcow API, not manual UI work.

### Tool email address registry

Every tool has exactly one owner, one purpose, one mailbox. Registry maintained alongside this ADR as the platform catalog:

| Tool | Mailbox | RX | TX | Purpose |
|---|---|---|---|---|
| GitLab CE | `gitlab@{domain}` | ❌ | ✅ | CI / MR / pipeline notifications |
| GitLab CE | `gitlab-incoming@{domain}` | ✅ | ❌ | Inbound issue / MR / Service Desk replies (`+token` subaddressing) |
| Grafana | `grafana@{domain}` | ❌ | ✅ | Alert emails |
| HashiCorp Vault | `vault@{domain}` | ❌ | ✅ | Seal / unseal / expiry alerts |
| Authentik | `authentik@{domain}` | ❌ | ✅ | User invites, password reset, MFA |
| NetBox | `netbox@{domain}` | ❌ | ✅ | Change notifications, webhook alerts |
| Nextcloud | `nextcloud@{domain}` | ❌ | ✅ | Share notifications, user invites |
| Vaultwarden | `vaultwarden@{domain}` | ❌ | ✅ | User invites, emergency access |
| Zabbix | `zabbix@{domain}` | ❌ | ✅ | Monitoring alerts |
| Platform | `noreply@{domain}` | ❌ | ✅ | Generic transactional no-reply sender |
| Platform | `admin@{domain}` | ✅ | ✅ | Platform admin — human-monitored |
| Platform | `catchall@{domain}` | ✅ | ✅ | Virtual address consumer (mandatory) |

**Tools with no email requirement** (not in the registry): OPNsense, Traefik, Pi-hole, PostgreSQL, Redis, step-ca, GitLab Runner, Prometheus, Loki. These either log to Loki or are alerted on by Grafana.

### Credentials

All mailbox IMAP/SMTP credentials live in HashiCorp Vault at paths defined by `security/0001-secret-storage`:

| Secret | Vault path |
|---|---|
| Per-tool mailbox (IMAP + SMTP password) | `secret/{env}/mail/{toolname}` |
| Catch-all mailbox | `secret/{env}/mail/catchall` |
| Mailcow API key (for Ansible provisioning) | `secret/{env}/mailcow/api-key` |
| M365 Graph API token (hybrid only) | `secret/{env}/exchange/graph-api-token` |

### Operational procedures (runbook, not ADR)

The following procedures live in `platform-setup/tools/mailcow/runbooks/lifecycle.md`, not in this ADR:

- Adding a new tool mailbox
- Adding a virtual address
- Inbound subaddressing flow
- Outbound from tool flow
- New human user onboarding (hybrid model)
- User offboarding (hybrid model)

ADRs define decisions; runbooks define procedures. Each runbook references this ADR for the decision it implements.

## Consequences

- **Zero per-tool M365 licenses.** Every tool mailbox is on Mailcow regardless of whether the org has Exchange.
- **Adding a tool mailbox is one Ansible task + one Vault entry.** No Exchange admin, no Mail Flow Rule.
- **Virtual addresses (`sales@`, `accounting@`) require zero infrastructure change** — the tool manages them internally via the catch-all consumer pattern.
- **Two connectors replace N per-tool Mail Flow Rules** in the hybrid model. Connector 1 and Connector 2 cover every inbound and outbound tool flow.
- **Mailcow is an official image deployment** — no custom builds, no external DB reconfiguration. Stays on the supported upgrade path.
- **Deployment pattern is the generic Linux VM pipeline** — Mailcow is not a provisioning exception like OPNsense.

## Revision triggers

Revise this ADR when:
- Mailcow is replaced by a different self-hosted mail server (Postfix + Dovecot standalone, iRedMail, Mailu, etc.)
- M365 is replaced by Google Workspace as the hybrid upstream — Connector model changes
- A third deployment model is introduced (e.g. fully cloud-hosted via Fastmail / SimpleLogin for small deployments)
- Per-user mailboxes become required on the platform (currently all human users go through Authentik, not email)
- DMARC alignment or DKIM rotation rules change
- The "Relay non-existing mailboxes only" setting is no longer sufficient as open-relay protection

## CISO mapping

| Framework | Controls covered |
|---|---|
| ISO 27001:2022 | A.5.14 (information transfer — authenticated SMTP / IMAP+SSL only, no plaintext), A.8.24 (use of cryptography — TLS on every mail leg), A.8.15 (logging — Mailcow + Exchange audit logs forwarded to Loki), A.9.4.2 (secure logon — IMAP creds in Vault, Connector 2 auth via cert) |
| NIS2 | Art. 21(2)(e) (network and information systems security — relay restricted, SPF enforced, no open relay) |
| GDPR | Art. 32(1)(a) (encryption of personal data in transit — TLS on every mail path) |
