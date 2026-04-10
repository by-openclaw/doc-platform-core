# ADR-0033: User Identity & Access Standard

**Status:** Accepted
**Date:** 2026-04-10
**Deciders:** @yboujraf

## Context

The platform uses service accounts (svc-rune) and local admin accounts (by-systems) across multiple VMs, LXCs, and services. No formal standard exists for user creation, SSH key management, GPG signing, sudo configuration, or how Authentik (future OIDC/LDAP authority) interacts with local accounts. A prior reference implementation exists but is application-specific and missing user creation, sudo, and agent autoload.

## Decision

### 1. Three Account Types

| Account | Type | Purpose | Source | Always available? |
|---|---|---|---|---|
| `svc-rune` | Service account | Day-to-day interactive ops, API, git | Authentik (future), local (now) | Depends on Authentik |
| `svc-ansible` | Automation account | Ansible, CI/CD, non-interactive | Local | YES |
| `by-systems` | Local admin | OOB emergency, break-glass, console | Always local (`/etc/passwd`) | YES — even if Authentik/network down |

**Rules:**
- Never delete the local admin. `by-systems` is the last resort.
- `svc-rune` = interactive (human at keyboard). `svc-ansible` = automation (no human).
- Separate accounts = separate blast radius. Compromised CI token does not give interactive access.

### 2. SSH Keys

| Rule | Requirement |
|---|---|
| Algorithm | ED25519 only. No RSA, no ECDSA. |
| Passphrase | Mandatory. No empty passphrase keys. |
| One key per user | One ED25519 key pair per user identity. Not per machine. |
| Key comment | `{username}@{domain}` (e.g. `svc-rune@by-systems.be`) — agnostic, not tied to hostname |
| Agent | ssh-agent loaded in `.bashrc`. Passphrase typed once per session. |
| Storage | Private key: local `~/.ssh/` + Vault KV backup. Public key: git repo `infra/keys/`. |

### 3. GPG Keys

| Rule | Requirement |
|---|---|
| Algorithm | EdDSA (Ed25519) |
| Passphrase | Mandatory |
| Identity | `{username} <{username}@{domain}>` |
| Expiry | 365 days. Renewal alert 30 days before expiry. |
| Agent | gpg-agent loaded in `.bashrc`. `GPG_TTY=$(tty)`. |
| Storage | Private key: Vault KV. Public key: git repo + GitHub. |
| Git signing | All commits and tags signed (`commit.gpgsign=true`, `tag.gpgsign=true`) |

### 4. Sudo

**Rule: `NOPASSWD` is FORBIDDEN.** Every sudo requires a password. No exceptions.

| Account | Sudo | Password source | Config |
|---|---|---|---|
| `svc-rune` | Password required | User types it (interactive) | `/etc/sudoers.d/svc-rune` |
| `svc-ansible` | Password required | `ansible_become_pass` from Vault KV | `/etc/sudoers.d/svc-ansible` |
| `by-systems` | Password required | User types it (console/OOB) | `/etc/sudoers.d/by-systems` |

#### Automation (Ansible, CI/CD)

Non-interactive automation cannot type a password. The become password is stored in HashiCorp Vault and injected at runtime:

```yaml
# group_vars/all.yml (ansible-vault encrypted or Vault lookup)
ansible_become_pass: "{{ lookup('hashi_vault', 'secret/automation/svc-ansible:become_pass') }}"
```

A dedicated `svc-ansible` service account is used for automation:
- Separate from `svc-rune` (interactive ops) — blast radius isolation
- Password stored in Vault KV, never on disk
- Sudo is password-required (Ansible provides it via `become_pass`)
- SSH key with passphrase (Ansible uses `ssh-agent` or Vault SSH secrets engine)

#### Sudoers config

```
# /etc/sudoers.d/svc-rune
svc-rune ALL=(ALL:ALL) ALL

# /etc/sudoers.d/svc-ansible
svc-ansible ALL=(ALL:ALL) ALL

# /etc/sudoers.d/by-systems
by-systems ALL=(ALL:ALL) ALL
```

No `NOPASSWD`. No `!authenticate`. Every privilege escalation is audited.

### 5. Shell

| Account | Linux | OPNsense |
|---|---|---|
| `svc-rune` | `/bin/bash` | `/bin/sh` (mandatory for SSH — default "none" kills connection) |
| `by-systems` | `/bin/bash` | `/bin/sh` |

### 6. Groups

| Group | Members | Purpose |
|---|---|---|
| `sshuser` | svc-rune, by-systems | SSH AllowGroups (sshd hardening) |
| `sudo` | svc-rune, by-systems | Sudo access |
| `docker` | svc-rune | Docker operations (where applicable) |

### 7. Agent Autoload (`.bashrc`)

```bash
# SSH agent — passphrase once per session
if [ -z "$SSH_AUTH_SOCK" ]; then
    eval "$(ssh-agent -s)" > /dev/null
    ssh-add ~/.ssh/id_ed25519 2>/dev/null
fi

# GPG agent
export GPG_TTY=$(tty)
```

### 8. Provisioning

Six concerns, executed in order by Ansible:

| # | Concern | Ansible role | Runs on |
|---|---|---|---|
| 1 | User / group creation + sudo | `roles/user-mgmt` | All VMs/LXCs |
| 2 | sshd hardening (port, crypto, socket disable) | `roles/sshd-hardening` | All VMs/LXCs |
| 3 | Banner (`/etc/issue.net`) | `roles/banner` | All VMs/LXCs |
| 4 | Git config (system + per-user) | `roles/git-config` | Dev VMs only |
| 5 | SSH + GPG keys (generate, upload, Vault) | `roles/key-mgmt` | Dev VMs only |
| 6 | fail2ban (SSH jail, OOB whitelist) | `roles/fail2ban` | All VMs/LXCs |

### 9. Authentik Integration (Future)

| Phase | State |
|---|---|
| Now | Local users, Ansible provisioned |
| Phase 2 | Authentik deployed, LDAP provider, SSSD on Linux VMs |
| Phase 3 | svc-rune sourced from Authentik. by-systems stays local (OOB fallback). |

### 10. Secret Storage

| Secret | Location (now) | Location (target) |
|---|---|---|
| SSH private key | `~/.ssh/id_ed25519` | Same + Vault KV backup |
| SSH passphrase | User memory | Vault KV |
| GPG private key | `~/.gnupg/` | Same + Vault KV backup |
| GPG passphrase | User memory | Vault KV |
| SSH public key | git repo `infra/keys/` | Same |
| GPG public key | git repo + GitHub | Same |
| API tokens | `infra/secrets/*.env` | Vault KV |

## Consequences

- Every new VM/LXC runs the 6-role provisioning playbook
- No more ad-hoc user creation or SSH key copying
- One key per user, passphrase mandatory — existing keyless keys must be rotated
- OPNsense users MUST have `shell=/bin/sh` set (discovered 2026-04-10)
- by-systems local account is never managed by Authentik — always local fallback
- 6 existing SSH keys on Rune VM need consolidation to 1

## CISO mapping

### ISO/IEC 27001:2022

| Control | Title | Status | Notes |
|---|---|---|---|
| A.5.15 | Access control | Covered | Two account types, least privilege, group-based |
| A.5.17 | Authentication information | Covered | ED25519 + passphrase, no empty keys |
| A.8.5 | Secure authentication | Covered | SSH key + passphrase, agent-based |

### NIS2 (Directive 2022/2555)

| Article | Requirement | Status | Notes |
|---|---|---|---|
| Art. 21(2)(d) | Supply chain security | Covered | Signed commits, GPG verification |
| Art. 21(2)(i) | Human resources security | Covered | Named accounts, no shared credentials, audit trail |
| Art. 21(2)(j) | Multi-factor auth | Partial | SSH key + passphrase = 2 factors. Authentik adds OIDC MFA later. |

### GDPR (Regulation 2016/679)

| Article | Requirement | Status | Notes |
|---|---|---|---|
| Art. 5(1)(f) | Integrity and confidentiality | Covered | Encrypted keys (passphrase), no plaintext credentials on disk |
| Art. 25 | Data protection by design | Covered | Least privilege accounts, group-based access, local fallback isolated |
| Art. 32 | Security of processing | Covered | SSH key + passphrase, agent-based auth, break-glass audit trail |

## References

- ADR-0010: Naming & Identity Convention
- ADR-0012: Environment Tier Standard
- ADR-0031: CI Token & Identity Standard
