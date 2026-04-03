# ADR-0026: Email Infrastructure Standard

**Date:** 2026-04-02
**Status:** Accepted
**Deciders:** yboujraf, Rune
**Extends:** ADR-0025 (notification/collaboration), ADR-0016 (Vault paths)

---

## Context

The platform needs a structured, provider-aware email infrastructure that:
- Receives inbound email for all platform addresses under `@{domain}`
- Routes each address to the correct tool mailbox
- Falls back to a catch-all mailbox for addresses not explicitly routed
- Provides outbound SMTP for all tools
- Requires zero per-tool M365 licenses

Two layers are used together:
- **M365** — public MX, inbound router (Mail Flow Rules), outbound SMTP relay
- **Mailcow** — internal mail host, stores all tool mailboxes and the catch-all

---

## Architecture

### Inbound (RX)

M365 is the public MX. It receives all inbound email for `@{domain}`.
Mail Flow Rules route each recipient address to its target mailbox on Mailcow.
**Rule LAST** catches anything not explicitly routed — forwarded to the catch-all mailbox.

```
External sender → <anyaddress>@{domain}
        │
        ▼
M365 — public MX (Internal Relay mode)
Mail Flow Rules — evaluated top to bottom, first match wins

  Rule 1: recipient = toolA@{domain}
          → forward to toolA@{mailcow-host}
          → Mailcow delivers to toolA mailbox
          → Tool A reads via IMAP

  Rule 2: recipient = toolB+*@{domain}    ← subaddressing supported
          → forward to toolB@{mailcow-host}
          → Mailcow delivers to toolB mailbox
          → Tool B reads via IMAP, extracts +token for internal routing

  Rule N: one rule per tool that needs inbound

  Rule LAST: recipient matches no rule above
          → forward to catchall@{mailcow-host}
          → Mailcow delivers to catch-all mailbox
          → Any tool reading catch-all inspects original To: field
          → Routes internally based on To: value
```

### Why Internal Relay is required

M365 domain type must be set to **Internal Relay** for `{domain}`.
Without this, M365 bounces unknown recipients (those not in Exchange) before any Mail Flow Rule fires.
Internal Relay suppresses the bounce and passes the message through the rules.

### Outbound (TX)

```
Tool sends email (notification, reply, alert)
        │
        ▼
Mailcow SMTP
  → authenticated per-mailbox (credential from Vault)
        │
        ▼
M365 SMTP relay connector
  → validates: sender IP in allowlist? SPF valid?
        │
        ▼
Delivered to external recipient
From: tool@{domain}
```

M365 relay connector is IP-based — no per-tool credential on M365 side.
SPF record includes Mailcow server IP(s).

---

## Catch-all mailbox

The catch-all is a regular Mailcow mailbox: `catchall@{domain}` (or any alias).

Any tool that needs to receive email for addresses that are **not** explicitly routed reads this mailbox via IMAP.
The tool inspects the original `To:` field in the raw message to determine routing.

```
Rule LAST → catchall mailbox
        │
        ▼
Tool polls catchall via IMAP
        │
        ▼
Reads original To: field
        │
      ┌─┴───────────────────────────────┐
      ▼                                 ▼
To: sales@{domain}            To: accounting@{domain}
→ route to sales queue        → route to accounting queue
```

The catch-all tool defines its own internal routing logic — the email infrastructure is not involved.
Virtual addresses (addresses that exist only inside the tool, never in M365 or Mailcow) are managed entirely within the tool.

---

## Mail Flow Rules — full set

Rules are ordered. Lower number = higher priority. First match wins, stop processing.

| Priority | Rule name | Condition | Action |
|---|---|---|---|
| 10 | Tool A inbound | recipient = toolA@ | forward to toolA@{mailcow-host} |
| 20 | Tool B inbound | recipient = toolB+*@ | forward to toolB@{mailcow-host} |
| ... | ... | ... | ... |
| 900 | Catch-all | Apply to all remaining | forward to catchall@{mailcow-host} |

**Rule 900 exception:** recipient is external (NotInOrganization) — do not redirect.
**Rule 900 exception:** recipient is member of Dynamic All Users group — deliver normally to their M365 mailbox.

---

## Dynamic Distribution Group: "Dynamic All Users"

Auto-includes: all Exchange-licensed mailboxes + all mail-enabled M365 groups.

Used exclusively as the exception set for Rule LAST (Rule 900).
Licensed M365 users receive their email directly — they never hit the catch-all.

Tool mailboxes are **not** in this group (they are not Exchange objects).
Tool email is handled by their explicit rule (Rule 10, 20, etc.), before Rule 900 ever fires.

---

## Mailbox inventory

All tool mailboxes live on Mailcow. Naming convention: `{tool}@{domain}`.

| Mailbox | Inbound (RX) | Outbound (TX) | Notes |
|---|---|---|---|
| `toolA@{domain}` | ✅ Explicit rule | ✅ Mailcow SMTP | IMAP credential in Vault |
| `toolB@{domain}` | ✅ Explicit rule + subaddressing | ✅ Mailcow SMTP | IMAP credential in Vault |
| `noreply@{domain}` | ❌ Not monitored | ✅ Mailcow SMTP | Send-only alias |
| `catchall@{domain}` | ✅ Rule LAST only | ✅ Mailcow SMTP | Read by catch-all consumer tool |

All credentials: Vault at `secret/{env}/mail/{toolname}`.

---

## Subaddressing (plus addressing)

M365: enable org-wide via one PowerShell command (`Set-OrganizationConfig -AllowPlusAddressInRecipients $true`).
Mailcow: supported natively, no configuration needed.

Enables tools to use `tool+{token}@{domain}` for internal thread/ticket routing.
M365 Mail Flow Rule matches on `tool+*@{domain}` — delivers to the same `tool@{mailcow}` mailbox.
Tool reads the `+token` from the `To:` field and routes internally.

---

## Outbound relay configuration (M365)

| Setting | Value |
|---|---|
| Connector type | Your organization's email server → Office 365 |
| Allowed senders | IP allowlist: Mailcow server IP(s) |
| SPF record | `v=spf1 ip4:{mailcow_ip} include:spf.protection.outlook.com ~all` |
| Anti-spam policy | Mailcow IPs added to M365 connection filter allowlist |
| SMTP port | 25 (direct relay) or 587 (authenticated) |

---

## Email account lifecycle flows

### Flow 1 — Add a new tool mailbox

```
1. Ansible mailcow.yml adapter: create mailbox toolC@{domain} on Mailcow
2. Ansible exchange.yml adapter: add Mail Flow Rule (priority N)
      recipient = toolC@ → forward to toolC@{mailcow-host}
3. Tool credentials written to Vault: secret/{env}/mail/toolC
4. Tool deployment reads credentials from Vault → configures IMAP + SMTP
```

No manual Exchange admin. No M365 license. Zero downtime — rule activates immediately.

---

### Flow 2 — Add a virtual address (tool-internal, no mailbox)

```
1. Admin adds virtual address inside the tool (alias, team, queue, etc.)
2. Nothing changes in M365 or Mailcow
3. External sender emails virtualaddr@{domain}
4. M365: no explicit rule matches → Rule LAST (900) fires
5. Forward to catchall@{mailcow-host}
6. Tool reading catch-all sees To: virtualaddr@{domain}
7. Tool routes internally to correct queue/team/object
```

Zero infrastructure change. Tool manages its own virtual address routing.

---

### Flow 3 — Inbound with subaddressing (thread routing)

```
1. Tool creates unique token per thread: abc123
2. Tool sets reply-to: toolB+abc123@{domain}
3. External user replies
4. M365 Rule 20 matches: toolB+*@ → forward to toolB@{mailcow-host}
5. Mailcow delivers to toolB mailbox
6. Tool polls toolB mailbox via IMAP
7. Tool extracts +abc123 from To: field
8. Tool appends reply to correct thread/ticket
```

---

### Flow 4 — Outbound notification

```
1. Tool triggers notification/alert/reply
2. Tool connects to Mailcow SMTP (credential from Vault)
3. Mailcow relays to M365 SMTP connector
4. M365 validates sender IP → accepts
5. M365 delivers to external recipient
6. Recipient sees: From: toolA@{domain}
```

---

### Flow 5 — New human user

```
1. Admin creates user in identity provider (ADR-0024 flow)
2. Ansible exchange.yml: create licensed M365 mailbox user@{domain}
3. User auto-added to Dynamic All Users group
4. Email to user@{domain} → delivered by M365 directly (not via rules)
5. User configures mail client via M365 autodiscover
```

---

### Flow 6 — User offboarding

```
1. Admin disables user in identity provider
2. Ansible exchange.yml: disable M365 mailbox (not deleted — audit trail preserved)
3. User removed from Dynamic All Users group
4. Email to user@{domain} → Rule LAST → catch-all
5. Hard deletion: explicit manual action after audit confirmation (ADR-0024 policy)
```

---

## TX/RX matrix

| Mailbox type | Receives (RX) | Sends (TX) | How tool reads | How tool sends |
|---|---|---|---|---|
| M365 licensed user | ✅ Direct (M365) | ✅ Outlook / SMTP | Outlook / IMAP | Outlook / SMTP |
| Mailcow tool mailbox | ✅ Forwarded from M365 rule | ✅ Mailcow SMTP → M365 relay | IMAP + password (Vault) | SMTP + password (Vault) |
| Mailcow catch-all | ✅ Rule LAST only | ✅ Mailcow SMTP → M365 relay | IMAP + password (Vault) | SMTP + password (Vault) |
| Virtual address (tool-internal) | ❌ Not a real mailbox | ✅ Tool sends as alias | N/A — tool internal | Tool SMTP via Mailcow |

---

## Mailcow — configuration standard

| Property | Value |
|---|---|
| Role | Internal mail host for all tool mailboxes |
| Domain | `{domain}` — same domain as M365 |
| MX | Not public — M365 is the only public MX |
| Certs | step-ca (internal) or LE (if Mailcow web UI is public-facing) |
| API | REST API, Bearer token |
| Credentials | Vault at `secret/{env}/mailcow/api-key` |
| Catch-all | One catch-all alias per domain, points to `catchall@{domain}` mailbox |
| Subaddressing | Native, no config needed |
| DKIM | Configured per domain via Mailcow API |

### Mailcow API — supported operations

| Object | CRUD |
|---|---|
| Domains | ✅ |
| Mailboxes | ✅ |
| Aliases (including catch-all) | ✅ |
| DKIM records | ✅ |
| Quota management | ✅ |

---

## Ansible provisioning adapters

```
roles/identity-sync/adapters/
  exchange.yml    ← M365: Mail Flow Rules, Dynamic All Users group, user mailboxes
  mailcow.yml     ← Mailcow: domains, mailboxes, aliases, catch-all, DKIM

playbooks/identity-sync/
  exchange.yml
  mailcow.yml
```

Both adapters are driven by the same deployment manifest (`mail_provider` var selects adapter path).

---

## Deployment variables

```yaml
# deployments/{org}-{env}/deployment.yml
org:
  domain: by-systems.be
  env: prod
  mail_provider: m365        # m365 | mailcow | both
  mailcow_host: mail.by-systems.be
  mailcow_ip: 10.6.225.80
```

---

## Email client autoconfiguration (internal users on Mailcow)

Pi-hole provides DNS records for zero-touch mail client setup:

```
autoconfig.{domain}   → Mailcow IP   (Thunderbird)
autodiscover.{domain} → Mailcow IP   (Outlook)
_imap._tcp.{domain}   → SRV record
_smtp._tcp.{domain}   → SRV record
```

---

## Credentials summary

| Secret | Vault path |
|---|---|
| Tool mailbox IMAP/SMTP | `secret/{env}/mail/{toolname}` |
| Catch-all mailbox | `secret/{env}/mail/catchall` |
| Mailcow API key | `secret/{env}/mailcow/api-key` |
| M365 Graph API (exchange adapter) | `secret/{env}/msteams/graph-api` |

---

## Consequences

- Zero per-tool M365 licenses — all tool mailboxes on Mailcow
- Adding a tool mailbox = one Mailcow mailbox + one M365 Mail Flow Rule (both via Ansible)
- Virtual addresses = zero infrastructure change — tool manages internally
- M365 is the inbound router only — not a mail store for tools
- All credentials in Vault — never in config files or environment variables at rest
- Subaddressing works on both M365 and Mailcow — no provider lock-in for thread routing

---

## CISO mapping

### ISO/IEC 27001:2022

| Control | Title | Status | Notes |
|---|---|---|---|
| A.8.24 | Use of cryptography | ✓ | TLS on all SMTP relay; IMAP over SSL only |
| A.5.14 | Information transfer | ✓ | Authenticated SMTP or IMAP+SSL; no plaintext |
| A.8.15 | Logging | ✓ | M365 mail flow audit logging; Mailcow logs to Loki (future) |
| A.9.4.2 | Secure logon procedures | ✓ | IMAP credentials in Vault; M365 relay is IP-based |

### NIS2 (Directive 2022/2555)

| Article | Requirement | Status | Notes |
|---|---|---|---|
| Art. 21(2)(e) | Security in network and information systems | ✓ | Relay restricted by IP allowlist; SPF enforced; no open relay |

---

## References

- ADR-0025: Notification & Collaboration Integration
- ADR-0024: Identity Provisioning & Sync (Ansible adapters)
- ADR-0016: Vault KV path convention
- ADR-0014: Certificate strategy
- ADR-0010 §10: `{domain}` deployment variable
- `docs/references/m365-odoo-email-configuration.md` — ventor.tech guide (M365 catchall setup)
- Tool-specific email configs: `platform-setup/tools/{tool}/docs/setup.md`
