# identity/0004 — OS Accounts & Host-Level Identity

**Status:** Draft
**Date:** 2026-04-12 (supersedes flat ADR-0033, 2026-04-10)
**Scope:** Linux/OPNsense host accounts, SSH/GPG keys, sudo, break-glass access.
**Related:** `identity/0001-authentication`, `security/0001-secret-storage`, `security/0003-hardening`

---

## Context

The platform runs service accounts (`svc-rune`), automation accounts (`svc-ansible`), and local admins (`by-systems`) across many VMs, LXCs and appliances. Without a standard, SSH key hygiene, sudo config, and break-glass access drift per host.

## Decision

### 1. Three account types

| Account | Type | Purpose | Source | Always available |
|---|---|---|---|---|
| `svc-rune-{env}` | Service | Interactive ops, API, git | Authentik (future), local (now) | Depends on Authentik |
| `svc-ansible-{env}` | Automation | Ansible, CI/CD, non-interactive | Local | ✅ |
| `{org}` | Local admin | OOB/break-glass, console | Always local (`/etc/passwd`) | ✅ |

**Names are authoritative in [`naming/0002-identity`](../naming/0002-identity.md)** — service accounts `svc-{function}-{env}` (§3), primary groups `{org}`/`rune`/`ansible` and the role groups (§5), and the `{org}` break-glass account. **There is no `sshuser` group** (§5.3). This ADR governs account **lifecycle**, not names; any account/group names in the prose below are illustrative.

**Rules:**
- `{org}` is never deleted — last-resort break-glass. Teardown is **disable, never delete**.
- `svc-rune-{env}` (human at keyboard) is separate from `svc-ansible-{env}` (automation). Compromised CI token must not give interactive access.

### 2. SSH keys

| Rule | Value |
|---|---|
| Algorithm | **ED25519 only** (no RSA, no ECDSA) |
| Passphrase | **Mandatory** (no empty passphrases) |
| Key per user | One ED25519 pair per identity, not per machine |
| Comment | `{username}@{domain}` (e.g. `svc-rune@by-systems.be`) — host-agnostic |
| Agent | `ssh-agent` loaded in `.bashrc`; passphrase once per session |
| Storage | Private: `~/.ssh/` + HashiCorp Vault KV backup. Public: git `infra/keys/` |
| Rotation | **365 days.** SSH keys have no native expiry; rotation is enforced by Ansible (`key-mgmt` role) using key age stored as Vault KV metadata. Renewal alert 30 days before deadline. |

### 3. GPG keys

| Rule | Value |
|---|---|
| Algorithm | **EdDSA (Ed25519)** |
| Passphrase | Mandatory |
| Identity | `{username} <{username}@{domain}>` |
| Expiry | 365 days; renewal alert 30 days before |
| Git signing | `commit.gpgsign=true`, `tag.gpgsign=true` — all commits and tags signed |
| Storage | Private: Vault KV. Public: git + GitHub |

### 4. Sudo

**`NOPASSWD` is forbidden.** Every sudo requires a password.

| Account | Password source |
|---|---|
| `svc-rune` | User types it (interactive) |
| `svc-ansible` | `ansible_become_pass` from Vault KV |
| `by-systems` | User types it (console/OOB) |

```
# /etc/sudoers.d/{svc-rune,svc-ansible,by-systems}
{account} ALL=(ALL:ALL) ALL
```

No `NOPASSWD`, no `!authenticate`. Every escalation is audited.

### 5. SSH break-glass (password auth from OOB CIDR)

Password authentication is **disabled globally** (`PasswordAuthentication no`). Password auth is re-enabled **only** via a `Match` block for the break-glass group from the OOB management network.

```
# /etc/ssh/sshd_config (templated by ansible-platform/roles/hardening)
PasswordAuthentication no
PubkeyAuthentication  yes

Match Group {{ hardening_ssh_break_glass_group }} Address {{ hardening_ssh_break_glass_networks | join(',') }}
    PasswordAuthentication yes
```

**Variables:**
- `hardening_ssh_break_glass_group` — POSIX group of accounts allowed to use password fallback (e.g. `break-glass`)
- `hardening_ssh_break_glass_networks` — CIDR list of the OOB management network

**Rules:**
- Password auth is only ever reachable from the OOB VLAN.
- Only members of the break-glass group can use it.
- Regular SSH (public-key, non-OOB) is unaffected.
- fail2ban protects this block with a short ban window and OOB whitelist for trusted ops stations.

Implementation: `ansible-platform/roles/hardening/templates/sshd_config.j2`.

### 6. Shell

| Account | Linux | OPNsense |
|---|---|---|
| `svc-rune` | `/bin/bash` | `/bin/sh` (mandatory — default "none" kills SSH) |
| `by-systems` | `/bin/bash` | `/bin/sh` |

### 7. Groups

Group **names and membership** are authoritative in [`naming/0002-identity §5`](../naming/0002-identity.md) — not duplicated here. sshd `AllowGroups` uses the **bare primary groups** (`{org}`, `rune`, and `ansible` once `svc-ansible-{env}` is provisioned) plus `break-glass` (§5.1/§5.3) — **there is no `sshuser` group.** `sudo`, `break-glass`, and `docker` are role groups created by Ansible (§5.2). This section adds only the lifecycle rule below.

**`docker` group rules:**
- Group membership is created **only on VMs where Docker is installed** (Ansible `user-mgmt` role, conditional on `docker_installed` fact).
- **Only `svc-rune`** is added. `{org}` is **not** added — docker group = root equivalence, which conflicts with the least-privilege break-glass role.
- `{org}` can still run docker in emergencies via `sudo docker ...` (slower, audited, acceptable for break-glass).

### 8. Agent autoload (`.bashrc`)

```bash
# SSH agent — passphrase once per session
if [ -z "$SSH_AUTH_SOCK" ]; then
    eval "$(ssh-agent -s)" > /dev/null
    ssh-add ~/.ssh/id_ed25519 2>/dev/null
fi
# GPG agent
export GPG_TTY=$(tty)
```

### 9. Provisioning (Ansible roles, in order)

| # | Concern | Role | Scope |
|---|---|---|---|
| 1 | User/group creation + sudo | `user-mgmt` | All VMs/LXCs |
| 2 | sshd hardening (incl. break-glass Match block) | `sshd-hardening` | All VMs/LXCs |
| 3 | Banner (`/etc/issue.net`) | `banner` | All VMs/LXCs |
| 4 | Git config (system + per-user) | `git-config` | Dev VMs only |
| 5 | SSH + GPG keys (generate, upload, Vault) | `key-mgmt` | Dev VMs only |
| 6 | fail2ban (SSH jail, OOB whitelist) | `fail2ban` | All VMs/LXCs |

### 10. Authentik integration (phased)

| Phase | State |
|---|---|
| Now | Local users, Ansible provisioned |
| Phase 2 | Authentik deployed, LDAP provider, SSSD on Linux VMs |
| Phase 3 | `svc-rune` sourced from Authentik. `by-systems` stays local (OOB fallback). |

### 11. Secret storage

**All "Vault" references in this ADR mean HashiCorp Vault** — the programmatic machine-secret store (KV engine, policies, dynamic secrets). Read and written by Ansible, CI, and applications.

> **Not Vaultwarden.** Vaultwarden is a user-facing password manager (Bitwarden-compatible) for humans remembering which password opens which web app (Grafana, NetBox, Proxmox WebUI, etc.). It is out of scope for this ADR and covered separately under `services/` when deployed.

| Secret | Now | Target |
|---|---|---|
| SSH private key | `~/.ssh/id_ed25519` | Same + HashiCorp Vault KV backup |
| SSH passphrase | User memory | HashiCorp Vault KV |
| SSH key age metadata | n/a | HashiCorp Vault KV (used by `key-mgmt` rotation) |
| GPG private key | `~/.gnupg/` | Same + HashiCorp Vault KV backup |
| GPG passphrase | User memory | HashiCorp Vault KV |
| SSH public key | git `infra/keys/` | Same |
| GPG public key | git + GitHub | Same |
| API tokens | `infra/secrets/*.env` | HashiCorp Vault KV |

## Consequences

- Every new VM/LXC runs the 6-role provisioning playbook
- No ad-hoc user creation or SSH key copying
- One key per user, passphrase mandatory — existing keyless keys must be rotated
- OPNsense users **must** have `shell=/bin/sh` (default "none" breaks SSH)
- `by-systems` is never managed by Authentik — always local fallback
- Password SSH is only reachable from the OOB VLAN, only for the break-glass group

## CISO mapping

| Framework | Controls covered |
|---|---|
| ISO 27001:2022 | A.5.15, A.5.17, A.8.5 |
| NIS2 | Art. 21(2)(d), 21(2)(i), 21(2)(j) (⚠ partial — MFA via Authentik future) |
| GDPR | Art. 5(1)(f), Art. 25, Art. 32 |
