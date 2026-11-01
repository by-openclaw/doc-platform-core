# identity/0005 — Remote Access Session Expiry

**Status:** Draft (proposal — approve by merging)
**Date:** 2026-10-31
**Scope:** How long an interactive remote-access session (a NetBird peer authenticated through Authentik) stays valid before the person must authenticate again, and which peers are outside that rule. Does not define who may reach what (NetBird policies derive from the service register) nor Authentik's own session lengths.
**Related:** `identity/0001-authentication`, `security/0001-secret-storage`, `security/0002-compliance-mapping`, `security/0003-hardening`, `infra/0001-platform-stack`

---

## Context

- Every person reaches the platform through NetBird; the peer logs in through Authentik (OIDC) with TOTP enforced (`identity/0001`). NetBird's *peer login expiration* forces that login again after a fixed period. Vendor documentation ("Enforce periodic user authentication"): default 24 hours, allowed range 1 hour to 180 days, applies only to peers added by interactive SSO login — peers enrolled with a setup key are not affected — and an expired peer shows `Login required` and stays disconnected until the person authenticates again.
- The platform role carries the vendor default (`netbird_peer_login_expiration: 86400`, comment "tune for NIS2 later"): the value was inherited, never decided.
- 2026-10-30: every client had expired overnight; the people read it as "the VPN is down" and the platform was audited for an outage that did not exist. A daily re-login also trains people to click through authentication prompts, which weakens MFA instead of strengthening it.
- NIS2 and ISO 27001 ask for a documented, risk-based re-authentication rule; neither prescribes a number.

## Decision

**Interactive remote-access sessions expire after 7 days (604 800 s); every expiry requires a fresh Authentik login with TOTP; peers enrolled with a setup key (service peers) are the only exception and are outside the rule by product design.**

| Rule | Value | Enforced by |
|---|---|---|
| Peer login expiration (SSO peers) | 7 days — `netbird_peer_login_expiration: 604800` | `roles/netbird` (ansible-platform), account settings pushed on every run |
| Expiration enabled | always — `netbird_peer_login_expiration_enabled: true` | same |
| Per-peer "never expire" on a person's device | forbidden; a peer found with it is drift and is reset by the role | `roles/netbird` peer review (to implement with this ADR) |
| Service / headless peers | setup key, no expiry (product behaviour); one key per service, named after it, stored in Vault | `roles/netbird`, `security/0001` |
| Re-authentication factor | Authentik OIDC + TOTP, no password-only path | `identity/0001` |
| Lost or compromised device | peer deleted in NetBird (immediate), the person's Authentik sessions revoked | `security/0003` runbook |

Why 7 days: one working week bounds the time a stolen, unlocked laptop keeps network access without a second factor, while the re-login happens once a week instead of at a random hour every day. Shorter periods (8 h, 24 h) produced support noise and false outage reports; longer ones (30–180 days) exceed what the hardening baseline accepts for an unattended credential (`security/0003`).

## Consequences

- People authenticate once a week; desktop clients (0.75 and later) warn 10 minutes before expiry.
- The role is the single place the number lives; the register's `vpn` exposure class inherits it.
- A peer left in `Login required` for more than 7 days is picked up by the periodic access review, not alerted.
- No change for automation: service peers enrol with setup keys.

## Alternatives considered

| Option | Rejected because |
|---|---|
| Keep 24 h (vendor default) | daily re-login = daily "VPN down" reports and MFA fatigue |
| 8 h (one shift) | several re-logins a day for people working across time zones; the same fatigue, more of it |
| 30 days or more | an unlocked stolen device keeps access for a month without a second factor |
| Disable expiry, rely on the Authentik session | the NetBird tunnel stays up regardless of the IdP session; the control would not exist |

## Revision triggers

- NetBird device posture checks are adopted: expiry may lengthen for compliant managed devices.
- A security incident involving a remote-access device.
- Belgian NIS2 implementing guidance fixes a value.

## CISO mapping

| Framework | Controls covered |
|---|---|
| ISO 27001:2022 | A.5.15 (access control), A.5.16 (identity management), A.5.17 (authentication information), A.5.18 (access rights — periodic re-validation), A.8.5 (secure authentication — MFA at every re-login) |
| NIS2 | Art. 21(2)(i) (access control policies), Art. 21(2)(j) (multi-factor authentication and secured communications) |
