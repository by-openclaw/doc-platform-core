# Reference: M365 + Odoo Email Configuration

**Source:** https://ventor.tech/guides/how-to-configure-emails-to-work-with-office-365-and-odoo/
**Saved:** 2026-04-02
**Applied in:** ADR-0026 (Email Infrastructure Standard)

This guide is the basis for the M365 + Odoo email integration used in BY-SYSTEMS deployments.
The platform implementation is defined in ADR-0026. This document is the upstream reference — consult it for step-by-step Exchange admin UI configuration.

## Summary of what this guide covers

1. **Outgoing mail (Odoo → external)**
   - M365 SMTP relay connector (Your org's email server → Office 365)
   - IP allowlist on relay connector (whitelist Odoo server IP)
   - Anti-spam connection filter policy: add Odoo IPs
   - SPF DNS record: `v=spf1 ip4:{odoo_ip} include:spf.protection.outlook.com ~all`

2. **Catchall mailbox**
   - Create licensed M365 user: `catchall@{domain}` (Exchange Online license required — not a shared mailbox)
   - Enable IMAP/SMTP/POP connections on this user

3. **Dynamic Distribution Group: "Dynamic All Users"**
   - Type: Dynamic distribution
   - Members: Users with Exchange mailboxes + Mail-enabled groups
   - Alias: `alldynamicusers@{domain}`
   - Purpose: Exception list for catchall rule — real users/groups are NOT redirected to Odoo

4. **Mail Flow Rule: "Catchall for Odoo"**
   - Apply to: All messages
   - Action: Redirect to catchall mailbox
   - Exceptions:
     - Recipient is external (NotInOrganization)
     - Recipient is member of Dynamic All Users group
   - Mode: Enforce
   - Priority: High
   - Stop processing more rules: Yes

5. **Domain → Internal Relay**
   - Exchange Admin → Mail flow → Accepted domains → set domain type to "Internal Relay"
   - Without this: M365 bounces email to unknown addresses before Mail Flow Rule can act

6. **Azure App registration for Odoo IMAP access**
   - Required for OAuth 2.0 / Graph API connection from Odoo to catchall mailbox
   - Scopes: IMAP.AccessAsUser.All (or equivalent)

## How Odoo virtual addresses work

Odoo defines virtual addresses internally (Sales Teams, Projects, Support queues).
These addresses do NOT exist in Exchange. When someone emails `sales@{domain}`:
1. M365 sees an unknown recipient
2. Domain is Internal Relay → no bounce
3. Mail Flow Rule fires → redirects to `catchall@{domain}`
4. Odoo reads catchall mailbox via IMAP/Graph
5. Odoo matches the original `To:` address against its internal routing rules
6. Odoo routes to the correct Sales Team / Project / Support Queue

Odoo virtual addresses are created/managed entirely in Odoo. No Exchange admin required.

## Platform tool mailboxes (our addition — not in ventor.tech guide)

Platform tool shared mailboxes (`gitlab@{domain}`, `netbox@{domain}`, etc.) are real Exchange shared mailboxes. They appear in the Dynamic All Users group automatically and are therefore **exempt** from the catchall redirect rule. They receive email normally.

See ADR-0026 for the full platform email architecture.
