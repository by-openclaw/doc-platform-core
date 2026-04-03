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

## M365 + Odoo inbound routing model

### Architecture

```
Inbound to @{domain}
        │
        ▼
M365 Mail Flow Rule: "Catchall for Odoo"
  ├── Recipient is in Dynamic All Users group?
  │     (real licensed mailboxes + known shared mailboxes)
  │     → deliver normally — rule does not apply
  │
  └── Recipient is anything else?
        (Odoo virtual addresses: sales@, support@, project-xyz@, etc.)
        → NOT a real M365 address
        → redirected to Odoo catchall mailbox
        → Odoo reads via IMAP/Graph API
        → routes to correct Sales Team / Project / Support Queue in Odoo
```

### Why this works

M365 domain type is set to **Internal Relay** for `{domain}`. Without this setting, M365 would bounce email to unknown addresses before the Mail Flow Rule can act. Internal Relay allows unknown recipients to pass through the rule.

### Dynamic Distribution Group: "Dynamic All Users"

Automatically includes: all Exchange-licensed mailboxes + all mail-enabled groups + shared mailboxes.

This group is the exception condition in the Mail Flow Rule. Any recipient who is a member of this group is **not** redirected to catchall — they receive email normally.

Platform tool mailboxes (`gitlab@{domain}`, `netbox@{domain}`, etc.) are Exchange shared mailboxes. They appear in the Dynamic All Users group automatically. Their email is never forwarded to Odoo catchall.

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

## Platform tool mailboxes (M365 shared)

Tool mailboxes are Exchange shared mailboxes. No per-tool license required (shared mailboxes are free up to 50 GB). Email is routed directly to them.

**The only access challenge is programmatic read.** Shared mailboxes have no password — tools cannot do a direct IMAP login. Two solutions exist, both requiring zero additional licenses.

### Tool email access model

| Tool | Inbound needed? | Access method | License? |
|---|---|---|---|
| GitLab | ✅ Yes (incoming_email, Service Desk) | Graph API app registration | None |
| NetBox | ❌ No — send only | SMTP relay (IP connector) | None |
| Grafana | ❌ No — send only | SMTP relay (IP connector) | None |
| Vault | ❌ No — send only | SMTP relay (IP connector) | None |
| Odoo | ✅ Yes (catchall) | Licensed mailbox (required by Odoo) | 1 × Exchange Online |

**Tools that only send** use the M365 SMTP relay connector (IP-allowlisted). No credential, no mailbox, no license.

**Tools that need to read** (currently only GitLab) use an Azure app registration with Graph API permissions on the shared mailbox — no license required.

**Naming convention:** `{tool}@{domain}`

| Mailbox | Type | Purpose |
|---|---|---|
| `gitlab@{domain}` | Shared | GitLab outbound notifications, system email |
| `gitlab-incoming@{domain}` | Shared | GitLab inbound (incoming_email, Service Desk) |
| `netbox@{domain}` | Shared | NetBox outbound notifications |
| `grafana@{domain}` | Shared | Grafana alert emails |
| `vault@{domain}` | Shared | Vault notification emails |
| `noreply@{domain}` | Shared | Transactional no-reply sender |
| `catchall@{domain}` | **Licensed** | Odoo inbound receiver (Exchange Online required) |

---

### GitLab inbound — Graph API (no license)

GitLab v16+ supports Microsoft Graph API natively for incoming email. This is the preferred method over IMAP basic auth (which M365 is deprecating).

**Azure app registration required:**
- Permissions: `Mail.Read`, `Mail.Send` scoped to `gitlab-incoming@{domain}` shared mailbox
- Credentials: Vault at `secret/{env}/gitlab/graph-api`

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

### Subaddressing (plus addressing)

M365 supports `+` subaddressing — enabled org-wide via one PowerShell command. Required for GitLab inbound email.

```
gitlab-incoming+{token}@{domain}  →  delivers to gitlab-incoming shared mailbox
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
