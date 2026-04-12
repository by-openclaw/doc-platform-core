# ADR-0026: Email Infrastructure Standard

**Date:** 2026-04-02
**Status:** Accepted
**Deciders:** yboujraf, Rune
**Extends:** ADR-0025 (notification/collaboration), ADR-0016 (Vault paths)

---

## Context

The platform needs a structured email infrastructure that:
- Receives inbound email for all platform addresses under `@{domain}`
- Routes to the correct tool mailbox on Mailcow, or falls back to catch-all
- Provides outbound SMTP for all tools through Exchange
- Requires zero per-tool M365 licenses

Two deployment phases:

| Phase | Mail provider | Exchange involved? | Purpose |
|---|---|---|---|
| **PoC** | Mailcow standalone | ❌ No | Validate stack in isolation. Prod is never touched. No third-party relay. |
| **Prod** | Mailcow + Exchange hybrid | ✅ Yes | Full model: Exchange as public MX + relay, Mailcow as mail host. |

**PoC decision:** Mailcow is the sole MX. All inbound and outbound goes through Mailcow directly. No Exchange connector, no relay dependency. Platform is fully self-contained.

**Migration trigger:** When PoC stack is validated end-to-end → add Exchange hybrid (two connectors) without changing any tool config. Tool mailboxes, credentials, and IMAP access remain identical.

Reference: https://docs.mailcow.email/third_party/exchange_onprem/third_party-exchange_onprem/

---

## Architecture

### Phase 1 — PoC: Mailcow standalone

```
External sender → <anyaddress>@{domain}
        │
        ▼
┌─────────────────────────────────────────────────────────────┐
│               Mailcow (sole MX)                             │
│               Public MX record → Mailcow IP                 │
│                                                             │
│  Known mailbox (gitlab@, grafana@, vault@, ...)?            │
│  └── YES → deliver to tool mailbox                          │
│            Tool reads via IMAP (credential from Vault)      │
│                                                             │
│  Unknown (virtual address)?                                 │
│  └── NO  → catchall@{domain}                                │
│            Consumer reads via IMAP, inspects To:            │
│                                                             │
│  Outbound: Tool → Mailcow SMTP → direct delivery            │
│            No relay. No third party.                        │
└─────────────────────────────────────────────────────────────┘
```

No Exchange. No relay dependency. Prod is never touched.

---

### Phase 2 — Prod: Mailcow + Exchange hybrid (target)

### Inbound (RX) + Outbound (TX)

```
External sender → <anyaddress>@{domain}
        │
        ▼
┌─────────────────────────────────────────────────────────────┐
│               M365 Exchange (public MX)                     │
│               Domain type: Internal Relay                   │
│                                                             │
│  Known Exchange mailbox (licensed user)?                    │
│  └── YES → deliver directly to user Exchange mailbox        │
│                                                             │
│  Unknown recipient (tool address or virtual address)?       │
│  └── NO  → Connector 1: Exchange → Mailcow                  │
└──────────────────────────────┬──────────────────────────────┘
                               │ Connector 1 (inbound relay)
                               ▼
┌─────────────────────────────────────────────────────────────┐
│               Mailcow (internal mail host)                  │
│               Forwarding host: Exchange gateway             │
│               Mode: Relay non-existing mailboxes only       │
│                                                             │
│  Known Mailcow mailbox (toolA@, toolB@, ...)?               │
│  └── YES → deliver to tool mailbox                          │
│            Tool reads via IMAP (credential from Vault)      │
│                                                             │
│  Unknown (virtual address: sales@, accounting@, ...)?       │
│  └── NO  → catch-all mailbox on Mailcow                     │
│            Consumer reads via IMAP                          │
│            Inspects original To: → routes internally        │
│                                                             │
│  Outbound (tool sends email):                               │
│  Tool → Mailcow SMTP → relayhost = Exchange gateway         │
└──────────────────────────────┬──────────────────────────────┘
                               │ Connector 2 (outbound relay)
                               ▼
┌─────────────────────────────────────────────────────────────┐
│               M365 Exchange (outbound relay)                │
│               Auth: TLS cert or static IP                   │
│               Delivers to external recipient                │
│               From: tool@{domain}                           │
└─────────────────────────────────────────────────────────────┘
```

---

## Two Exchange connectors

| Connector | Direction | Purpose |
|---|---|---|
| Connector 1 | Exchange → Mailcow | Route unknown recipients to Mailcow |
| Connector 2 | Mailcow → Exchange | Allow Mailcow to relay outbound via Exchange |

**Connector 2 authentication:** TLS certificate (subject = accepted domain in Exchange) or static IP of Mailcow server. TLS cert is the recommended method.

---

## M365 configuration

| Setting | Value |
|---|---|
| Domain type | **Internal Relay** (required — otherwise M365 bounces unknowns before Connector 1 fires) |
| MX record | `{org}.mail.protection.outlook.com` |
| Mail Flow Rules | None per tool — two connectors handle everything |
| Dynamic All Users group | Includes all licensed Exchange mailboxes — excluded from Connector 1 routing |

---

## Mailcow configuration

| Setting | Value |
|---|---|
| Relayhost | Exchange personalized gateway (e.g. `contoso-com.mail.protection.outlook.com`) |
| Forwarding host | Same Exchange gateway — accept relay from Exchange unconditionally |
| Domain relay mode | Relay this domain ✅, Relay all recipients ✅, **Relay non-existing mailboxes only** ✅ |
| Catch-all | `catchall@{domain}` mailbox — receives anything with no matching Mailcow mailbox |
| Subaddressing | Native, no config needed |
| DKIM | Configured per domain via Mailcow API |

**"Relay non-existing mailboxes only"** is the critical setting: Mailcow only accepts relay for addresses it has no mailbox for. Prevents open relay. Unknown addresses go to catch-all.

---

## Catch-all mailbox

Regular Mailcow mailbox: `catchall@{domain}`.

Receives all email for virtual addresses — addresses that exist only inside a tool (not in Exchange, not in Mailcow).

```
catchall@{domain} receives email for sales@, accounting@, project-xyz@, ...
        │
        ▼
Consumer tool polls via IMAP
        │
        ▼
Reads original To: field from raw message
        │
      ┌─┴──────────────────────────────────┐
      ▼                                    ▼
To: sales@{domain}             To: accounting@{domain}
→ route to sales queue         → route to accounting queue
```

Virtual addresses are defined and managed entirely inside the tool. Zero Exchange or Mailcow admin to add/remove them.

---

## Subaddressing (plus addressing)

Enables tools to use `tool+{token}@{domain}` for thread/ticket routing.

- **M365:** `Set-OrganizationConfig -AllowPlusAddressInRecipients $true`
- **Mailcow:** native, no configuration needed

```
toolB+abc123@{domain}
  → Exchange: unknown recipient → Connector 1 → Mailcow
  → Mailcow: toolB mailbox exists → deliver there
  → Tool reads To: field, extracts +abc123 → routes to correct thread
```

---

## Mailbox inventory

All tool mailboxes on Mailcow. Naming: `{tool}@{domain}`.

| Mailbox | RX | TX | Notes |
|---|---|---|---|
| `toolA@{domain}` | ✅ Via Connector 1 | ✅ Mailcow SMTP → Connector 2 | IMAP credential in Vault |
| `toolB@{domain}` | ✅ Via Connector 1 + subaddressing | ✅ Mailcow SMTP → Connector 2 | IMAP credential in Vault |
| `noreply@{domain}` | ❌ Not monitored | ✅ Mailcow SMTP → Connector 2 | Send-only alias |
| `catchall@{domain}` | ✅ Fallback (no Mailcow mailbox match) | ✅ Mailcow SMTP → Connector 2 | Read by catch-all consumer |

All credentials: Vault at `secret/{env}/mail/{toolname}`.

---

## Lifecycle flows

### Flow 1 — Add a new tool mailbox

```
1. Ansible mailcow.yml: create mailbox toolC@{domain} on Mailcow
2. Credentials written to Vault: secret/{env}/mail/toolC
3. Tool deployment reads credentials from Vault → configures IMAP + SMTP
```

No Exchange admin. No M365 license. No Mail Flow Rule. Mailcow routing is automatic.

---

### Flow 2 — Add a virtual address (tool-internal only)

```
1. Admin adds virtual address inside the tool (alias, team, queue, etc.)
2. Nothing changes in Exchange or Mailcow
3. External sender emails virtualaddr@{domain}
4. Exchange: unknown → Connector 1 → Mailcow
5. Mailcow: no mailbox for virtualaddr → catch-all
6. Consumer reads catch-all, sees To: virtualaddr@{domain} → routes internally
```

Zero infrastructure change.

---

### Flow 3 — Inbound with subaddressing

```
1. Tool generates token per thread: abc123
2. Tool sets reply-to: toolB+abc123@{domain}
3. External user replies
4. Exchange: unknown → Connector 1 → Mailcow
5. Mailcow: toolB mailbox exists → deliver there
6. Tool polls toolB via IMAP, extracts +abc123 → routes to correct thread
```

---

### Flow 4 — Outbound from tool

```
1. Tool sends via Mailcow SMTP (credential from Vault)
2. Mailcow relays via relayhost (Exchange gateway, Connector 2)
3. Exchange delivers to external recipient
4. From: tool@{domain}
```

---

### Flow 5 — New human user

```
1. Admin creates user in identity provider (ADR-0024)
2. Ansible exchange.yml: create licensed M365 mailbox user@{domain}
3. User added to Dynamic All Users group → delivered by Exchange directly
4. User configures mail client via M365 autodiscover
```

---

### Flow 6 — User offboarding

```
1. Admin disables user in identity provider
2. Ansible exchange.yml: disable Exchange mailbox (not deleted — audit trail)
3. User removed from Dynamic All Users group
4. Email to user@{domain} → no Exchange mailbox → Connector 1 → Mailcow catch-all
5. Hard deletion: explicit manual action after audit confirmation (ADR-0024 policy)
```

---

## TX/RX matrix

| Mailbox type | RX | TX | How tool reads | How tool sends |
|---|---|---|---|---|
| M365 licensed user | ✅ Exchange direct | ✅ Outlook / SMTP | Outlook / IMAP | Outlook / SMTP |
| Mailcow tool mailbox | ✅ Via Connector 1 | ✅ Mailcow → Connector 2 | IMAP + password (Vault) | SMTP + password (Vault) |
| Mailcow catch-all | ✅ Fallback (no mailbox match) | ✅ Mailcow → Connector 2 | IMAP + password (Vault) | SMTP + password (Vault) |
| Virtual address | ❌ Not a real mailbox | ✅ Tool sends as alias | N/A — tool internal | Tool SMTP via Mailcow |

---

## Mailcow — API operations

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
  exchange.yml    ← M365: licensed mailboxes, Dynamic All Users group, Connector config
  mailcow.yml     ← Mailcow: domains, mailboxes, aliases, catch-all, DKIM, relayhost

playbooks/identity-sync/
  exchange.yml
  mailcow.yml
```

---

## Deployment variables

```yaml
# PoC
org:
  domain: by-systems.be
  env: poc
  mail_provider: mailcow-only    # Mailcow is sole MX, no Exchange
  mailcow_host: mail.poc.by-systems.be
  mailcow_ip: 10.1.3.55

# Prod (target — when PoC validated)
org:
  domain: by-systems.be
  env: prod
  mail_provider: hybrid          # Mailcow + Exchange two-connector model
  exchange_gateway: contoso-com.mail.protection.outlook.com
  mailcow_host: mail.by-systems.be
  mailcow_ip: 10.6.225.80
```

---

## Email client autoconfiguration

Pi-hole DNS records for zero-touch client setup:

```
autoconfig.{domain}   → Mailcow IP   (Thunderbird)
autodiscover.{domain} → Mailcow IP   (Outlook, for tool accounts)
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
- Adding a tool mailbox = one Mailcow mailbox + Vault entry (no Exchange config)
- Virtual addresses = zero infrastructure change — tool manages internally
- Two connectors replace all per-tool Mail Flow Rules — simpler, less admin
- All credentials in Vault — never in config files

---

## CISO mapping

### ISO/IEC 27001:2022

| Control | Title | Status | Notes |
|---|---|---|---|
| A.8.24 | Use of cryptography | ✓ | TLS on Connector 2; IMAP over SSL only |
| A.5.14 | Information transfer | ✓ | Authenticated SMTP or IMAP+SSL; no plaintext |
| A.8.15 | Logging | ✓ | M365 mail flow audit logging; Mailcow logs to Loki (future) |
| A.9.4.2 | Secure logon procedures | ✓ | IMAP credentials in Vault; Connector 2 auth via TLS cert or static IP |

### NIS2 (Directive 2022/2555)

| Article | Requirement | Status | Notes |
|---|---|---|---|
| Art. 21(2)(e) | Security in network and information systems | ✓ | Relay restricted by IP/TLS; SPF enforced; no open relay |

---

## References

- Mailcow Exchange Hybrid setup: https://docs.mailcow.email/third_party/exchange_onprem/third_party-exchange_onprem/
- Mailcow relayhost guide: https://docs.mailcow.email/manual-guides/Postfix/u_e-postfix-relayhost/
- Microsoft connector setup: https://docs.microsoft.com/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/set-up-connectors-to-route-mail
- ADR-0025: Notification & Collaboration Integration
- ADR-0024: Identity Provisioning & Sync
- ADR-0016: Vault KV path convention
- ADR-0014: Certificate strategy
- ADR-0010 §10: `{domain}` deployment variable
- Tool-specific email configs: `platform-setup/tools/{tool}/docs/setup.md`

---

## Tool email address registry

One mailbox per tool on Mailcow. No ambiguity — each address has exactly one owner, one purpose.

Naming rule: `{tool}@{domain}` for send-only / general. `{tool}-incoming@{domain}` only when the tool needs a dedicated inbound with subaddressing (+token routing).

| Tool | Mailbox | RX | TX | Purpose |
|---|---|---|---|---|
| GitLab | `gitlab@{domain}` | ❌ | ✅ | System notifications (CI, MR, pipeline) — send-only |
| GitLab | `gitlab-incoming@{domain}` | ✅ | ❌ | Inbound replies to issues/MRs/Service Desk via +token subaddressing |
| Grafana | `grafana@{domain}` | ❌ | ✅ | Alert emails — send-only |
| Vault | `vault@{domain}` | ❌ | ✅ | Seal/unseal alerts, expiry notifications — send-only |
| Authentik | `authentik@{domain}` | ❌ | ✅ | User invites, password reset, MFA — send-only |
| NetBox | `netbox@{domain}` | ❌ | ✅ | Change notifications, webhook alerts — send-only |
| Nextcloud | `nextcloud@{domain}` | ❌ | ✅ | Share notifications, user invites — send-only |
| Nexus | `nexus@{domain}` | ❌ | ✅ | Artifact/repo alerts — send-only |
| Vaultwarden | `vaultwarden@{domain}` | ❌ | ✅ | User invites, emergency access — send-only |
| Odoo | `catchall@{domain}` | ✅ | ✅ | Catch-all consumer — virtual addresses (sales@, accounting@, etc.) |
| Odoo | `odoo@{domain}` | ❌ | ✅ | Odoo system outbound sender — send-only alias |
| Zabbix | `zabbix@{domain}` | ❌ | ✅ | Monitoring alerts — send-only |
| Unifi | `unifi@{domain}` | ❌ | ✅ | Network alerts — send-only |
| Platform | `noreply@{domain}` | ❌ | ✅ | Generic transactional no-reply sender |
| Platform | `admin@{domain}` | ✅ | ✅ | Platform admin — human-monitored |

**Total Mailcow mailboxes:** 15 (no license cost, no Exchange object for any of them)

### Tools with NO email requirement

The following tools do not send or receive email — excluded from the registry:

- OPNsense — syslog/SNMP, no email
- Traefik — logs to Loki, no email
- Pi-hole — no email
- PostgreSQL — no email (Grafana alerts cover DB metrics)
- Redis — no email
- step-ca — no email (cert expiry → Vault alerts)
- GitLab Runner — no email (GitLab handles notifications)
- Prometheus / Loki — Grafana handles alerting

---

## Reference: Mailcow Exchange Hybrid setup

```
External sender → <anyaddress>@{domain}
        │
        ▼
┌─────────────────────────────────────────────────────────────┐
│               M365 Exchange (public MX)                     │
│               Domain type: Internal Relay                   │
│                                                             │
│  Known Exchange mailbox (licensed user)?                    │
│  └── YES → deliver directly to user Exchange mailbox        │
│                                                             │
│  Unknown recipient (tool address or virtual address)?       │
│  └── NO  → Connector 1: Exchange → Mailcow                  │
└──────────────────────────────┬──────────────────────────────┘
                               │ Connector 1 (inbound relay)
                               ▼
┌─────────────────────────────────────────────────────────────┐
│               Mailcow (internal mail host)                  │
│               Forwarding host: Exchange gateway             │
│               Mode: Relay non-existing mailboxes only       │
│                                                             │
│  Known Mailcow mailbox (gitlab@, grafana@, vault@, ...)?    │
│  └── YES → deliver to tool mailbox                          │
│            Tool reads via IMAP (credential from Vault)      │
│                                                             │
│  Unknown (virtual address: sales@, accounting@, ...)?       │
│  └── NO  → catchall@{domain}                                │
│            Consumer reads via IMAP                          │
│            Inspects original To: → routes internally        │
│                                                             │
│  Outbound (tool sends email):                               │
│  Tool → Mailcow SMTP → relayhost = Exchange gateway         │
└──────────────────────────────┬──────────────────────────────┘
                               │ Connector 2 (outbound relay)
                               ▼
┌─────────────────────────────────────────────────────────────┐
│               M365 Exchange (outbound relay)                │
│               Auth: TLS cert or static IP                   │
│               Delivers to external recipient                │
│               From: tool@{domain}                           │
└─────────────────────────────────────────────────────────────┘
```

**Official source:** https://docs.mailcow.email/third_party/exchange_onprem/third_party-exchange_onprem/
**Microsoft connector guide:** https://docs.microsoft.com/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/set-up-connectors-to-route-mail
