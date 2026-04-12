# ADR-0004: Identity & SSO Architecture

**Date:** 2026-03-27
**Amended:** 2026-04-02
**Status:** Accepted
**Deciders:** yboujraf, Rune

---

## Context

The platform needs a unified SSO layer that:
- Works standalone, with no dependency on any external identity provider
- Optionally federates with external directories (EntraID, Google Workspace, LDAP) when the org has them
- Provides break-glass access when SSO is unavailable

---

## Decision

### Three-tier identity model

**Tier 1 — OS/App local accounts (OOB break-glass)**
- Every server: `root` + named local admin (e.g. `admin@pam` on Proxmox)
- No SSO dependency — always available
- Credentials: stored in HashiCorp Vault (when deployed), bootstrap in `infra/secrets/`
- Rule: never remove these accounts, never disable password auth without OOB alternative

**Tier 2 — Authentik local accounts (service accounts + Authentik-native users)**
- All CI/CD runners, Ansible service accounts, automation tokens
- External users (contractors, partners) without an org directory
- Managed via Authentik API / GitLab CI

**Tier 3 — Federated accounts (optional — org-dependent)**
- If the org has an external directory (EntraID, Google Workspace, LDAP), it can be connected as an upstream
- Authentik remains the platform IAM hub — external directory is a feed, not a dependency
- If external directory is down → Authentik local copy continues to work
- External directory sync is strictly opt-in, per org, per deployment

---

## Authentik is standalone — always

Authentik operates fully without any external identity provider.
External directory sync (AD, EntraID, Google Workspace, LDAP) is an **optional integration** — enabled per org on request, not a platform requirement.

```
Without external directory:          With external directory (optional):

  Authentik                            EntraID / Google / LDAP
  (manages all users natively)              │
       │                                    │ SCIM or LDAP sync (one-way)
       │ OIDC / SAML                        ▼
       ▼                              Authentik (platform IAM hub)
  Platform tools                           │
                                           │ OIDC / SAML
                                           ▼
                                      Platform tools
```

In both cases: platform tools only ever talk to Authentik. The upstream directory is invisible to them.

---

## Architecture

```
[Optional upstreams — one or more, per org]
  EntraID / Azure AD  ──SCIM 2.0──────────────────────┐
  Google Workspace    ──OIDC / Google Social Login─────┤
  On-prem AD / LDAP   ──LDAP sync──────────────────────┤
  GitHub              ──OAuth 2.0──────────────────────┤
  GitLab (self-hosted)──OIDC──────────────────────────►│
                                                        │
                                               Authentik (platform IAM hub)
                                               — always available
                                               — local auth always enabled
                                                        │
                                               OIDC / SAML
                                                        │
                                          ┌─────────────┴────────────┐
                                     GitLab CE                  NetBox
                                     HashiCorp Vault             Grafana
                                     Proxmox VE                  Teleport CE
                                     Guacamole                   NetBird
                                     (all platform tools)
```

---

## User population model

| User type | Identity source | Auth method | Managed by |
|---|---|---|---|
| Org employee (with EntraID/Google) | External directory | Social login (OIDC) + Authentik local fallback | External directory / IT |
| Org employee (no external directory) | Authentik-native | Username + Authentik password | Authentik admin |
| Customer / freelancer / partner | Authentik-native | Username + Authentik password | Authentik admin |
| Service account (CI, Ansible, automation) | Authentik-native | API token / client credentials | Authentik / Vault |
| Break-glass (emergency ops) | OS local | SSH key / console | Vault / infra team |

---

## External directory sync — optional, per org

When an org has an external directory and wants to sync it:

| Directory | Protocol | Sync direction | Group mapping |
|---|---|---|---|
| Microsoft EntraID | SCIM 2.0 | EntraID → Authentik (one-way) | EntraID group names → Authentik platform group names (mapping table in Ansible vars) |
| Google Workspace | OIDC / Google Social Login | Login-time only (no SCIM) | Manual group assignment in Authentik post-login |
| On-prem AD / LDAP | LDAP sync | AD → Authentik (one-way) | LDAP group DN → Authentik groups |
| GitHub / GitLab | OAuth 2.0 / OIDC | Login-time only | No group sync — Authentik groups assigned manually |

**Key rules:**
- Sync direction is always inbound to Authentik — never the reverse
- External directory down → Authentik local auth continues to work (non-blocking)
- Password sync: never — Authentik password is always independent
- MFA: enforced in Authentik regardless of upstream provider
- All users must complete Authentik enrollment (set local password) at first login — prevents lockout when upstream is unavailable

**Group naming bridge (EntraID):**
EntraID group names are controlled by the org's IT department and will not follow the Authentik platform naming convention. A mapping table is maintained in Ansible vars:
```yaml
entraid_group_mapping:
  "BY-SYSTEMS Platform Developers": tool-gitlab-developers
  "BY-SYSTEMS Platform Admins":     org-admins
  "BY-SYSTEMS GitLab Owners":       tool-gitlab-owners
```
Authentik only ever sees the mapped platform group names. EntraID naming is isolated to the sync adapter.

---

## Fallback behavior

| Scenario | Result |
|---|---|
| External directory down | Authentik local users continue to work normally |
| Authentik down | Break-glass: local OS accounts on each server |
| Both down | SSH key auth to each server (keys in `~/.ssh/authorized_keys`) |
| All auth down | Physical console / iLO access |

---

## Consequences

- Authentik is the only mandatory identity component. No external directory required.
- If external directory is connected: users have two credentials (org password + Authentik local). Authentik local is always the fallback.
- When Vault is deployed: Authentik service account credentials managed via Vault dynamic secrets
- MFA: enforced in Authentik (TOTP), independent of any upstream provider
- Authentik HA (Phase 2.5): 2-node deployment to eliminate Authentik as SPOF

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

## CISO mapping

> Applies only to controls directly relevant to this ADR's scope.

### ISO/IEC 27001:2022

| Control | Title | Status | Notes |
|---|---|---|---|
| A.5.15 | Access control | ✓ Covered | Unified SSO model with group-based access scope |
| A.5.16 | Identity management | ✓ Covered | Employees, external users, service accounts, and break-glass identities are explicitly defined |
| A.5.17 | Authentication information | ✓ Covered | MFA mandatory for external users; break-glass isolated to OS local accounts |
| A.8.2 | Privileged access rights | ✓ Covered | Break-glass accounts restricted; service accounts segregated from human users |
| A.8.15 | Logging | ⚠ Partial | Authentik audit log defined; Wazuh/Loki forwarding is future state |

### NIS2 (Directive 2022/2555)

| Article | Requirement | Status | Notes |
|---|---|---|---|
| Art. 21(2)(a) | Risk management | ✓ Covered | Identity tiers and break-glass flow reduce lockout and blast-radius risk |
| Art. 21(2)(i) | Human resources security | ✓ Covered | External and employee account lifecycle is explicitly managed |

### GDPR (Regulation 2016/679)

| Article | Requirement | Status | Notes |
|---|---|---|---|
| Art. 32 | Security of processing | ✓ Covered | MFA, scoped access, and audited logins support secure access control |
