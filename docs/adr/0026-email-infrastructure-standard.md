# ADR-0026: Email Infrastructure Standard

**Date:** 2026-04-02
**Status:** Accepted
**Deciders:** yboujraf, Rune
**Extends:** ADR-0025 (notification/collaboration), ADR-0016 (Vault paths)

---

## Context

The platform has two email tiers with distinct roles:

1. **M365 Exchange** — real user mailboxes, platform tool mailboxes, Odoo integration (production and any deployment with a real domain)
2. **Mailcow** — internal platform tool mailboxes, PoC/dev/test deployments where M365 is not in use

This ADR defines the architecture of both tiers, the inbound routing model, outbound relay configuration, the Odoo catchall integration, and the Ansible provisioning adapters.

---

## M365 inbound routing model

### Architecture

M365 acts as **inbound MX + router**. Mail Flow Rules route each address to the correct destination in priority order. Odoo catchall is the last resort — it only receives what no other rule matched.

```
Inbound to @{domain}
        │
        ▼
M365 Mail Flow Rules — evaluated top to bottom, first match wins

  Rule 1: recipient = gitlab-incoming@ OR gitlab-incoming+*@
          → deliver to gitlab-incoming shared mailbox (M365)
          → OR forward to gitlab-incoming@{mail-host} (Mailcow/external)
          → GitLab mail_room reads it, routes by +token

  Rule 2: recipient = netbox@{domain}
          → deliver to netbox shared mailbox

  Rule 3: recipient = grafana@{domain}
          → deliver to grafana shared mailbox

  Rule N: one rule per tool that receives inbound

  Rule LAST: recipient matches no rule above
             (Odoo virtual addresses: sales@, accounting@, support@, ...)
          → redirect to catchall@{domain}
          → Odoo reads catchall via IMAP
          → routes to correct Sales Team / Project / Accounting queue
```

### Why Internal Relay is required

M365 domain type must be set to **Internal Relay** for `{domain}`. Without this, M365 bounces email to unknown addresses (Odoo virtual addresses) before any Mail Flow Rule can act. Internal Relay passes them through to the rules.

### Routing to external destinations (non-M365 tools)

When a tool mailbox is not hosted on M365 (e.g. Mailcow for PoC tier), the Mail Flow Rule action is **forward to external address** instead of deliver to shared mailbox:

```
Rule 1: recipient = gitlab-incoming+*@by-systems.be
        → forward to gitlab-incoming@mail.poc.example.com  (Mailcow)

Rule 2: recipient = netbox@by-systems.be
        → forward to netbox@mail.poc.example.com  (Mailcow)

Rule LAST: no match → catchall@by-systems.be → Odoo
```

M365 is the inbound router only. Delivery destination can be any valid email address — M365 shared mailbox, Mailcow, or any external IMAP host.

### Dynamic Distribution Group: "Dynamic All Users"

Includes: all Exchange-licensed mailboxes + all mail-enabled groups + shared mailboxes.
Used as the exception list in the Rule LAST (catchall) to ensure licensed users never get redirected to Odoo.
Shared tool mailboxes appear in this group automatically — they are delivered by their explicit rule, not by Rule LAST.

---

## Odoo mailbox — licensed, not shared

Odoo requires a **licensed Exchange Online mailbox** as its catchall receiver. A shared mailbox (no license) is not sufficient — Odoo needs IMAP or Graph API access with full read/write permissions.

| Property | Value |
|---|---|
| Mailbox address | `catchall@{domain}` (or `odoo@{domain}`) |
| License | Exchange Online (minimum) |
| Role | User (no administrator access) |
| Purpose | Inbound receiver for all Odoo virtual addresses |
| Connection method | IMAP or Graph API (OAuth 2.0) |
| Credentials | Vault at `secret/{env}/odoo/catchall-mailbox` |

Odoo virtual addresses (Sales Teams, Projects, Support queues, Accounting) are **defined in Odoo only**. They are never created in Exchange. When someone emails one of these addresses, M365 does not know the address — it forwards to catchall — Odoo processes it and routes to the correct team/queue.

**Examples of Odoo virtual addresses (Odoo-only, never in Exchange):**
- `sales@{domain}` → Sales Team
- `support@{domain}` → Helpdesk queue
- `accounting@{domain}` → Accounting / Invoicing inbox
- `info@{domain}` → General enquiries
- `project-xyz@{domain}` → Project-specific inbox

Odoo virtual addresses are created/managed entirely in Odoo. No Exchange admin required.

---

## Outbound relay (Odoo → external)

Odoo sends email from virtual addresses (e.g., `from: support@{domain}` on behalf of a customer reply). M365 acts as SMTP relay.

### Configuration

| Setting | Value |
|---|---|
| Connector type | Your organization's email server → Office 365 |
| Allowed senders | IP allowlist: Odoo server IP(s) |
| SPF record | `v=spf1 ip4:{odoo_ip} include:spf.protection.outlook.com ~all` |
| Anti-spam | Odoo IPs added to M365 connection filter allowlist |
| SMTP port | 25 (direct relay) or 587 (authenticated SMTP) |

Credentials for authenticated SMTP: Vault at `secret/{env}/odoo/smtp`.

Reference: [ventor.tech/guides/how-to-configure-emails-to-work-with-office-365-and-odoo](https://ventor.tech/guides/how-to-configure-emails-to-work-with-office-365-and-odoo/)

---

## Catch-all routing pattern (provider-agnostic)

The inbound routing model is the same regardless of mail provider. Odoo reads a catch-all mailbox via IMAP and inspects the original `To:` field to route to the correct internal object.

```
External user sends to <anyaddress>@{domain}
        │
        ▼
Mail provider receives it
  ├── Known real address (john@{domain}, gitlab@{domain})?
  │     → deliver normally
  │
  └── Unknown address (sales@, accounting@, project-xyz@, ...)?
        → catch-all mailbox (catchall@{domain})
        → Odoo reads mailbox via IMAP
        → Odoo inspects original To: field
        → routes to correct object (Sales Team, Project, Accounting, ...)
```

Odoo is **provider-agnostic**. Any mail provider that supports catch-all routing works:

| Provider | Catch-all mechanism | Odoo reads via |
|---|---|---|
| M365 | Mail Flow Rule + Internal Relay domain | IMAP or Graph API |
| Google Workspace | Catch-all routing in Admin console | IMAP |
| Mailcow (PoC/internal) | Native catch-all alias per domain | IMAP |
| Any IMAP server | Catch-all alias | IMAP |

For **Mailcow**: set `catchall@{domain}` as the domain catch-all alias. Odoo reads via IMAP. Identical behaviour to M365, no app registration required.

---

## Platform tool mailboxes

Platform tools that **send notifications** (Grafana, NetBox, Vault, etc.) use the mail provider SMTP relay. They do not receive inbound email — inbound is not applicable to them.

Platform tools that **receive email** have a dedicated mailbox:

| Mailbox | Type | Purpose |
|---|---|---|
| `catchall@{domain}` | **Licensed** (M365) / mailbox (Mailcow) | Odoo inbound receiver |
| `gitlab-incoming@{domain}` | Shared (M365) / mailbox (Mailcow) | GitLab inbound (incoming_email, Service Desk) |
| `gitlab@{domain}` | Shared | GitLab outbound system notifications |
| `netbox@{domain}` | Shared | NetBox outbound notifications |
| `grafana@{domain}` | Shared | Grafana alert emails (send only) |
| `vault@{domain}` | Shared | Vault notification emails (send only) |
| `noreply@{domain}` | Shared | Transactional no-reply sender |

**M365 shared mailboxes:** free up to 50 GB, no per-tool license. Email routes directly to them. Tools access them via IMAP (with OAuth 2.0) or Graph API — not via password.

**Odoo catch-all mailbox on M365:** requires one Exchange Online license. Odoo needs authenticated IMAP/Graph access with read permissions. A shared mailbox (no license) is insufficient for Odoo's connection model.

---

### GitLab inbound — Graph API (M365)

GitLab v16+ supports Microsoft Graph API natively for incoming email. Preferred over IMAP basic auth (deprecated by Microsoft).

**Azure app registration:** `Mail.Read` + `Mail.Send` scoped to `gitlab-incoming@{domain}`.
Credentials: Vault at `secret/{env}/gitlab/graph-api`.

```ruby
# gitlab.rb — Microsoft Graph API incoming email
gitlab_rails['incoming_email_enabled']       = true
gitlab_rails['incoming_email_inbox_method']  = 'microsoft_graph'
gitlab_rails['incoming_email_address']       = 'gitlab-incoming+%{key}@{domain}'
gitlab_rails['incoming_email_mailbox_name']  = 'gitlab-incoming@{domain}'
gitlab_rails['incoming_email_tenant_id']     = ENV['AZURE_TENANT_ID']     # from Vault
gitlab_rails['incoming_email_client_id']     = ENV['AZURE_CLIENT_ID']     # from Vault
gitlab_rails['incoming_email_client_secret'] = ENV['AZURE_CLIENT_SECRET'] # from Vault
```

### GitLab inbound — IMAP (Mailcow / any provider)

```ruby
# gitlab.rb — IMAP incoming email (Mailcow or any IMAP provider)
gitlab_rails['incoming_email_enabled']  = true
gitlab_rails['incoming_email_address']  = 'gitlab-incoming+%{key}@{domain}'
gitlab_rails['incoming_email_email']    = 'gitlab-incoming@{domain}'
gitlab_rails['incoming_email_password'] = ENV['GITLAB_INCOMING_EMAIL_PASSWORD'] # from Vault
gitlab_rails['incoming_email_host']     = 'mail.{domain}'
gitlab_rails['incoming_email_port']     = 993
gitlab_rails['incoming_email_ssl']      = true
```

### Subaddressing (plus addressing)

Required for GitLab `incoming_email` to route replies to the correct issue/MR thread.

- **M365:** enable org-wide via one PowerShell command
- **Mailcow:** supported natively, no configuration needed

```
gitlab-incoming+{token}@{domain}  →  delivers to gitlab-incoming mailbox
```

GitLab routes by `+token` to the correct issue/MR thread or Service Desk queue.

---

## M365 Mail Flow Rules (full set)

| Rule name | Condition | Action | Exceptions |
|---|---|---|---|
| Catchall for Odoo | Apply to all messages | Redirect to `catchall@{domain}` | Recipient is external (NotInOrganization) OR recipient is member of Dynamic All Users |
| GitLab inbound | Recipient = `gitlab-incoming@` or `gitlab-incoming+*@` | Deliver to `gitlab-incoming` shared mailbox | — |

Stop Processing More Rules = enabled on Catchall rule. Priority: GitLab inbound rule runs first.

---

## Mailcow — PoC/internal tier

Mailcow is the email platform for PoC, dev, and test deployments where M365 is not in use.

| Property | Value |
|---|---|
| MX for `example.com` | Mailcow (sole MX — no M365 involved) |
| DNS | Pi-hole internal only — no public MX records |
| Certs | step-ca `*.example.com` (ADR-0014 Rule 1: IANA-reserved → step-ca) |
| API | REST API (Bearer token) |
| Credentials | Vault at `secret/poc/mailcow/api-key` |

### Mailcow API capabilities

| Object | API support |
|---|---|
| Domains | ✅ Full CRUD |
| Mailboxes | ✅ Full CRUD |
| Aliases (including catchall) | ✅ Full CRUD |
| Shared folders (Dovecot ACL) | ✅ |
| DKIM records | ✅ |
| Quota management | ✅ |

### Mailcow multi-domain

One Mailcow instance manages all non-prod domains. Domain is a parameter in the Ansible adapter — never hardcoded.

```
example.com     ← poc/dev/test (IANA-reserved, Pi-hole only)
{domain}        ← staging/acc (real domain, internal routing)
```

---

## Ansible provisioning adapters

Both M365 and Mailcow use the same `identity-sync` role from ADR-0024, with dedicated adapters.

```
roles/identity-sync/adapters/
  exchange.yml    ← M365 mailboxes, shared mailboxes, groups, Mail Flow Rules via Graph API
  mailcow.yml     ← Mailcow domains, mailboxes, aliases, DKIM via REST API

playbooks/identity-sync/
  exchange.yml
  mailcow.yml
```

**exchange.yml adapter provisions:**
- Shared mailboxes (`{tool}@{domain}`) — create/update
- Dynamic Distribution Group — maintain membership
- Plus addressing — enable org-wide if not already set
- Shared mailbox IMAP access for tools that need it (GitLab `incoming_email`)

**mailcow.yml adapter provisions:**
- Domain onboarding (DKIM, MX, quotas)
- Tool mailboxes per deployment
- Aliases and catchall per domain

---

## Deployment-time variables

All domain and server references use `{domain}` as resolved from the deployment manifest (ADR-0010 §10):

```yaml
# deployments/{org}-{env}/deployment.yml
org:
  domain: by-systems.be
  env: prod
  mail_provider: m365   # m365 | mailcow
  odoo_ip: 10.6.225.50
```

`mail_provider` selects which adapter runs. Same playbook, different execution path.

---

## Email client autoconfiguration

For Mailcow deployments, Pi-hole provides internal DNS records for email client autoconfiguration:

```
autoconfig.{domain}    → Mailcow IP   (Thunderbird autoconfig)
autodiscover.{domain}  → Mailcow IP   (Outlook autodiscover)
_imap._tcp.{domain}    → SRV record
_smtp._tcp.{domain}    → SRV record
```

User enters `user@{domain}` in Thunderbird or Outlook → autoconfig/autodiscover → fully configured with no manual input.

---

## Email account lifecycle flows

### Flow 1 — New platform tool mailbox (e.g. add `netbox@{domain}`)

```
1. Ansible identity-sync playbook runs (exchange.yml or mailcow.yml adapter)
2. Adapter creates shared mailbox / Mailcow mailbox: netbox@{domain}
3. Adapter adds mailbox to Dynamic All Users group (M365 only)
4. Adapter adds Mail Flow Rule: recipient = netbox@ → deliver to netbox mailbox
5. Tool configured with SMTP credentials from Vault (outbound only for most tools)
6. If tool needs inbound: Vault credential injected at deploy via env var
```

Result: email to `netbox@{domain}` routes to netbox mailbox. No manual Exchange admin.

---

### Flow 2 — New Odoo virtual address (e.g. add `accounting@{domain}`)

```
1. Admin creates alias / Sales Team / queue in Odoo UI
2. Odoo internally maps accounting@{domain} → Accounting team
3. Nothing changes in M365 or Mailcow
4. External sender emails accounting@{domain}
5. M365: no explicit rule matches → Rule LAST fires → forward to catchall mailbox
6. Odoo reads catchall, sees To: accounting@{domain}
7. Odoo routes to Accounting queue
```

Result: zero M365/Mailcow admin for Odoo virtual addresses. Odoo manages its own routing.

---

### Flow 3 — Inbound email to GitLab (issue reply, Service Desk)

```
1. GitLab generates unique token per issue/MR: abc123
2. GitLab sets reply-to: gitlab-incoming+abc123@{domain}
3. User replies to that email
4. M365 Rule 1 matches: gitlab-incoming+*@ → deliver to gitlab-incoming mailbox
5. GitLab mail_room polls mailbox (IMAP or Graph API)
6. mail_room extracts +token from To: field
7. mail_room posts reply as comment on correct issue/MR
```

---

### Flow 4 — Outbound from tool (notification email)

```
1. Tool (Grafana, NetBox, Vault, GitLab) triggers notification
2. Tool connects to M365 SMTP relay (port 587 or 25)
3. M365 relay connector validates: sender IP in allowlist?
4. Yes → M365 accepts and delivers on behalf of tool
5. Recipient receives email from grafana@{domain} / gitlab@{domain} / etc.
```

SPF record includes tool server IPs. No per-tool M365 license. No credential on SMTP relay (IP-based auth).

---

### Flow 5 — Add a new human user (M365 licensed)

```
1. Admin creates user in Authentik (ADR-0024 provisioning flow)
2. Authentik webhook → Ansible identity-sync
3. exchange.yml adapter: create M365 licensed mailbox user@{domain}
4. User added to Dynamic All Users group (auto-included as licensed mailbox)
5. M365 Mail Flow Rules: user is in Dynamic All Users → never hits Rule LAST
6. User receives email at user@{domain} directly
7. User configures Outlook/Thunderbird via autoconfig/autodiscover (Mailcow) or M365 autodiscover
```

---

### Flow 6 — Remove a user (offboarding)

```
1. Admin disables user in Authentik
2. Authentik webhook → Ansible identity-sync
3. exchange.yml adapter: disable M365 mailbox (not deleted)
4. User removed from Dynamic All Users group
5. Any email to user@{domain} → Rule LAST → catchall → Odoo (unmatched address)
6. Odoo creates a ticket for undeliverable-looking inbound
7. Hard deletion: explicit manual action after audit confirmation (ADR-0024 removal policy)
```

---

### TX/RX matrix per mailbox type

| Mailbox type | Receives (RX) | Sends (TX) | How tool reads | How tool sends |
|---|---|---|---|---|
| M365 licensed user | ✅ Direct delivery | ✅ Native Outlook/SMTP | Outlook / IMAP / Graph API | Outlook / SMTP |
| M365 shared (tool) | ✅ Direct delivery | ✅ Via relay connector | Graph API or IMAP+OAuth | SMTP relay (IP auth) |
| Mailcow tool mailbox | ✅ Direct / forwarded from M365 | ✅ Via SMTP | IMAP + password (Vault) | SMTP + password (Vault) |
| Odoo catchall (M365) | ✅ Rule LAST only | ✅ Via relay connector | Graph API or IMAP+OAuth | SMTP relay |
| Odoo catchall (Mailcow) | ✅ Rule LAST forward | ✅ SMTP | IMAP + password (Vault) | SMTP + password (Vault) |
| Odoo virtual address | ❌ Not a real mailbox | ✅ Odoo sends as alias | N/A — Odoo internal | Odoo SMTP via relay |

---

## Credentials summary

| Secret | Vault path |
|---|---|
| Odoo catchall mailbox (IMAP/Graph) | `secret/{env}/odoo/catchall-mailbox` |
| Odoo SMTP relay | `secret/{env}/odoo/smtp` |
| GitLab Graph API (incoming email) | `secret/{env}/gitlab/graph-api` |
| M365 Graph API app credentials (Teams + shared mailboxes) | `secret/{env}/msteams/graph-api` |
| Mailcow API key | `secret/{env}/mailcow/api-key` |

---

## Consequences

- Odoo virtual addresses require no Exchange administration — add/remove entirely in Odoo
- Platform tool mailboxes require no Odoo involvement — direct Exchange shared mailboxes
- Adding a new tool mailbox = one Ansible task in exchange.yml
- Mailcow and M365 use identical playbook interface — `mail_provider` var selects adapter
- Email client autoconfiguration is zero-touch for internal deployments
- All IMAP/SMTP credentials in Vault — never in gitlab.rb or config files

---

## CISO mapping

> Applies only to controls directly relevant to this ADR's scope.

### ISO/IEC 27001:2022

| Control | Title | Status | Notes |
|---|---|---|---|
| A.8.24 | Use of cryptography | ✓ Covered | TLS on all SMTP relay connections; IMAP over SSL only |
| A.5.14 | Information transfer | ✓ Covered | All email flows use authenticated SMTP or IMAP+SSL; no plaintext |
| A.8.15 | Logging | ✓ Covered | M365 mail flow audit logging enabled; Mailcow logs to Loki (future) |
| A.9.4.2 | Secure logon procedures | ✓ Covered | IMAP access via OAuth 2.0 (Graph API) preferred; credentials in Vault |

### NIS2 (Directive 2022/2555)

| Article | Requirement | Status | Notes |
|---|---|---|---|
| Art. 21(2)(e) | Security in network and information systems | ✓ Covered | Relay connector restricts senders by IP; SPF enforced; no open relay |

---

## References

- ADR-0025: Notification & Collaboration Integration (notification routing, tool adapters)
- ADR-0016: Vault KV path convention
- ADR-0014: Certificate strategy (step-ca for Mailcow/internal)
- ADR-0010 §10: `{domain}` deployment variable
- ventor.tech guide: M365 + Odoo SMTP relay + catchall configuration
- Brainstorm: `brainstorming/2026-04-02-authentik-gitlab-identity-sync-brainstorm.md`
