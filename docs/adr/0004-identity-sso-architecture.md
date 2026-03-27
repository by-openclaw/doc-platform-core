# ADR-0004: Identity & SSO Architecture — Authentik + EntraID

**Date:** 2026-03-27
**Status:** Accepted
**Deciders:** yboujraf, Rune

---

## Context

The platform needs a unified SSO layer that:
- Works independently of cloud identity providers (resilience)
- Optionally federates with Microsoft EntraID (Azure AD) for org users
- Provides break-glass access when SSO is unavailable

---

## Decision

### Three-tier identity model

**Tier 1 — OS/App local accounts (OOB break-glass)**
- Every server: `root` + named local admin (e.g. `admin@pam` on Proxmox)
- No SSO dependency — always available
- Credentials: stored in HashiCorp Vault (when deployed), bootstrap in `infra/secrets/`
- Rule: never remove these accounts, never disable password auth without OOB alternative

**Tier 2 — Authentik local accounts (service accounts)**
- All CI/CD runners, Ansible service accounts, automation tokens
- Never federated with EntraID
- Managed via Authentik API / GitLab CI

**Tier 3 — EntraID + Authentik federated (human users)**
- Synced via SCIM 2.0: EntraID → Authentik
- Authentik stores local copy (username + local password set at first login)
- EntraID = source of truth for group membership and user lifecycle
- If EntraID is down → Authentik local copy continues to work
- If user deleted in EntraID → SCIM deactivates in Authentik
- Password sync: intentionally NOT synced (Authentik password is independent for resilience)

---

## Architecture

```
EntraID (Microsoft 365)
    │
    │ SCIM 2.0 (user/group sync, one-way push)
    ▼
Authentik (internal SSO broker — always available)
    │
    │ OIDC / SAML
    ▼
Platform services:
  GitLab CE → OIDC
  HashiCorp Vault → OIDC
  Proxmox VE → OIDC (future)
  NetBox → OIDC
  Grafana → OIDC
  Teleport CE → OIDC
  Guacamole → OIDC
  NetBird/WireGuard → OIDC
```

---

## EntraID federation (SCIM sync config)

| Setting | Value |
|---|---|
| Sync direction | EntraID → Authentik (one-way) |
| Protocol | SCIM 2.0 |
| Sync scope | Users in `BY-SYSTEMS Platform` group |
| Group mapping | EntraID groups → Authentik groups |
| Auto-provision | Authentik account created on first SCIM push |
| Auto-deprovision | Authentik account deactivated on EntraID delete |
| Password sync | None — Authentik password set independently at first login |

---

## Fallback behavior

| Scenario | Result |
|---|---|
| EntraID down | Authentik local users continue to work normally |
| Authentik down | Break-glass: local OS accounts on each server |
| Both down | SSH key auth to each server (keys in `~/.ssh/authorized_keys`) |
| All auth down | Physical console / iLO access |

---

## Consequences

- Users have two passwords: EntraID (for M365) and Authentik (for platform)
- Password rotation must be done in both systems independently
- When Vault is deployed: Authentik service account credentials managed via Vault dynamic secrets
- MFA: enforced in Authentik (TOTP), independent of EntraID MFA
- Authentik HA (Phase 2.5): 2-node deployment to eliminate Authentik as SPOF

---

## References
- Authentik SCIM source: https://docs.goauthentik.io/docs/providers/scim/
- NIS2 requirement: user lifecycle management, access revocation within 24h
- ISO 27001: A.9.2 (user access management), A.9.4 (system access control)

---

## Password flow clarification

**EntraID available (normal):**
- User clicks "Login with Microsoft" → redirected to EntraID → EntraID validates password → token returned to Authentik
- Authentik never sees or stores the EntraID password
- EntraID password changes take effect immediately with no sync needed

**EntraID unavailable (fallback):**
- User clicks "Login with password" → Authentik validates local password
- This is a separate credential set at first login via Authentik enrollment flow

**Risk mitigation:**
- All users MUST complete Authentik enrollment (set local password) at first login
- Enforced via Authentik enrollment flow — users cannot access platform services until local password is set
- Prevents lockout when EntraID is unavailable

---

## User population model

| User type | Identity source | Auth method | Managed by |
|---|---|---|---|
| BY-SYSTEMS employee | EntraID | "Login with Microsoft" (OIDC) | EntraID / IT |
| Customer / freelancer / partner | Authentik local | Username + Authentik password | Authentik admin |
| Service account (CI, Ansible, automation) | Authentik local | API token / client credentials | Authentik / Vault |
| Break-glass (emergency ops) | OS local | SSH key / console | Vault / infra team |

**Onboarding flows:**
- Employee: add to EntraID `BY-SYSTEMS Platform` group → SCIM pushes to Authentik → login via Microsoft
- Customer/freelancer: create Authentik account manually → invite email → set password → login with Authentik credentials
- No Microsoft 365 license required for external users

---

## External user provisioning policy

**Self-service signup: DISABLED**
No public registration. All accounts created by admin only.

**Customer onboarding flow:**
1. Admin creates account in Authentik (name, email, assign to customer group)
2. Authentik sends invite email with one-time enrollment link (via SMTP)
3. Customer sets own password + enrolls TOTP (MFA mandatory)
4. Customer accesses only services their group is authorized for

**Account lifecycle:**
- Active project → account enabled
- Project pause/end → account **disabled** (not deleted — audit trail preserved)
- Re-engagement → re-enable account
- Never delete accounts — disable only (NIS2/ISO 27001 audit trail requirement)

**Admin controls:**
- Per-account enabled/disabled flag
- Group membership controls access scope
- All login events logged (Authentik audit log → future: Wazuh/Loki)
