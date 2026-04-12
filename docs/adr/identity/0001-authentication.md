# identity/0001 — Authentication

**Status:** Draft
**Date:** 2026-04-12 (supersedes flat ADR-0004, 2026-03-27)
**Scope:** Platform IAM — who you are when you log in to any tool.
**Related:** `identity/0002-provisioning`, `identity/0003-machine-credentials`, `identity/0004-os-accounts`

---

## Decision

**Authentik is the only mandatory identity hub.** All platform tools authenticate through it. External directories (EntraID, Google, LDAP) are optional inbound feeds, never dependencies.

## Three-tier identity model

| Tier | Purpose | Always available |
|---|---|---|
| **1. OS/app local** | Break-glass (root, admin@pam) — per server | ✅ |
| **2. Authentik local** | Platform users, service accounts, external users | ✅ |
| **3. Federated (optional)** | Sync from EntraID / Google / LDAP | ❌ (opt-in per org) |

Rule: external directories are **inbound only**. Authentik local auth must always work when the upstream is down.

## Architecture

```
[Optional upstreams]              [Platform IAM hub]        [Tools]
 EntraID     ──SCIM 2.0──┐
 Google      ──OIDC──────┤──▶  Authentik  ──OIDC/SAML──▶  GitLab / NetBox
 LDAP        ──sync──────┤      (always               Vault / Grafana
 GitHub/Lab  ──OAuth─────┘       available)           Proxmox / etc.
```

Tools only ever talk to Authentik. The upstream directory is invisible to them.

## User populations

| User type | Source | Auth method |
|---|---|---|
| Employee (with external directory) | Upstream | Social login + Authentik local fallback |
| Employee (no directory) / customer / partner | Authentik-native | Username + password |
| Service account / automation | Authentik-native | API token / client credentials |
| Break-glass | OS local | SSH key / console |

**Rules:**
- Sync is always inbound (upstream → Authentik). Never reverse.
- Passwords never sync. Authentik password is always independent.
- MFA (TOTP) enforced in Authentik regardless of upstream.
- Every user completes Authentik enrollment at first login — prevents lockout.

## Fallback behavior

| Failure | Effect |
|---|---|
| External directory down | Authentik local auth keeps working |
| Authentik down | OS local break-glass — SSH key, or password auth from OOB CIDR (see `identity/0004-os-accounts` §SSH break-glass) |
| Network down | iLO / physical console |

## External user provisioning

- Self-service signup: **disabled**
- Admin creates account → Authentik sends enrollment link → user sets password + enrols TOTP
- Account lifecycle: **disable, never delete** (audit trail required by NIS2/ISO 27001)

## Consequences

- Authentik is the only mandatory identity component
- MFA is enforced centrally (TOTP), independent of upstream
- **Deployment:** Authentik runs as 2 instances (active/active HA) to eliminate SPOF

## CISO mapping

| Framework | Controls covered |
|---|---|
| ISO 27001:2022 | A.5.15, A.5.16, A.5.17, A.8.2, A.8.15 (⚠ partial — Loki forwarding future) |
| NIS2 | Art. 21(2)(a), 21(2)(i) |
| GDPR | Art. 32 |
