# git/0003 — Git Configuration Standard

**Status:** Draft
**Date:** 2026-04-12 (supersedes flat ADR-0034, 2026-04-10)
**Scope:** Host-level git configuration on all VMs/LXCs that run git.
**Related:** `git/0001-workflow`, `identity/0004-os-accounts` §GPG keys

---

## Context

Git config drifted across hosts: some had signing, others didn't. Line endings, default branches, and credential helpers varied. This ADR defines a standard applied by Ansible to every VM/LXC that uses git.

## Decision

### Two-layer configuration

| Layer | File | Scope | Managed by |
|---|---|---|---|
| System | `/etc/gitconfig` | All users on host | Ansible `roles/git-config` |
| User | `~/.gitconfig` | Per user | Ansible `roles/git-config` |

System layer = global invariants. User layer = identity and signing (per-user).

### System-wide properties (`/etc/gitconfig`)

| Property | Value | Reason |
|---|---|---|
| `core.autocrlf` | `input` | LF in repo, native on checkout |
| `core.eol` | `lf` | Force LF |
| `core.filemode` | `true` | Respect permissions |
| `core.quotePath` | `false` | Show Unicode paths |
| `init.defaultBranch` | `main` | Matches `git/0001-workflow` |
| `color.ui` | `auto` | Colored output |
| `protocol.version` | `2` | Git protocol v2 |
| `fetch.prune` | `true` | Clean stale remote branches |
| `fetch.fsckObjects` | `true` | Validate on fetch |
| `transfer.fsckObjects` | `true` | Validate on transfer |
| `http.sslVerify` | `true` | **Never** disable SSL |
| `pull.rebase` | `true` | Clean history |
| `push.default` | `current` | Push current branch |
| `push.autoSetupRemote` | `true` | Auto-track on first push |
| `push.followTags` | `true` | Tags follow commits |
| `merge.ff` | `false` | No fast-forward (matches `git/0001` merge strategy) |
| `merge.conflictStyle` | `zdiff3` | Better conflict markers |
| `diff.algorithm` | `histogram` | Better diffs |
| `diff.colorMoved` | `zebra` | Highlight moved code |
| `log.date` | `iso` | ISO date format |
| `log.abbrevCommit` | `true` | Shorter hashes |
| `credential.helper` | `cache --timeout=3600` | HTTPS token cache 1h |
| `filter.lfs.*` | LFS hooks | Git-LFS required |

### Per-user properties (`~/.gitconfig`)

| Property | Value | Reason |
|---|---|---|
| `user.name` | `svc-rune` or `by-systems` (etc.) | Commit author |
| `user.email` | `{username}@by-systems.be` | Commit email |
| `user.signingKey` | GPG fingerprint | See `identity/0004-os-accounts` §GPG |
| `user.useConfigOnly` | `true` | Prevent author guessing |
| `commit.gpgSign` | `true` | All commits signed |
| `tag.gpgSign` | `true` | All tags signed |
| `tag.forceSignAnnotated` | `true` | Force sign annotated tags |
| `gpg.program` | `gpg` | GPG binary |
| `gpg.format` | `openpgp` | Not x509 |

Commit/tag signing and GPG key lifecycle are defined in `identity/0004-os-accounts` §3. This ADR only enforces the git-side wiring.

### Templates (Jinja2, managed by `roles/git-config`)

**`/etc/gitconfig`:**

```jinja2
# {{ ansible_managed }}
[core]
    autocrlf = input
    eol = lf
    filemode = true
    quotePath = false
[init]
    defaultBranch = main
[color]
    ui = auto
[protocol]
    version = 2
[fetch]
    prune = true
    fsckObjects = true
[transfer]
    fsckObjects = true
[http]
    sslVerify = true
[pull]
    rebase = true
[push]
    default = current
    autoSetupRemote = true
    followTags = true
[merge]
    ff = false
    conflictStyle = zdiff3
[diff]
    algorithm = histogram
    colorMoved = zebra
[log]
    date = iso
    abbrevCommit = true
[credential]
    helper = cache --timeout=3600
[filter "lfs"]
    clean = git-lfs clean -- %f
    smudge = git-lfs smudge -- %f
    process = git-lfs filter-process
    required = true
```

**`~/.gitconfig`:**

```jinja2
# {{ ansible_managed }}
[user]
    name = {{ git_user_name }}
    email = {{ git_user_email }}
    signingKey = {{ gpg_key_id }}
    useConfigOnly = true
[commit]
    gpgSign = true
[tag]
    gpgSign = true
    forceSignAnnotated = true
[gpg]
    program = gpg
    format = openpgp
```

### What is NOT configured

| Property | Reason |
|---|---|
| `core.editor` | User preference |
| `alias.*` | User preference |
| `pager.*` | User preference |
| `ssh.variant` | Auto-detected |
| `http.proxy` | Environment-specific, not global |

### Applicability

| Host type | OS | System config | User config | Ansible transport |
|---|---|---|---|---|
| Dev/ops VMs | Linux | ✅ | ✅ | SSH |
| Service LXCs (no git use) | Linux | ✅ | ❌ | SSH |
| OPNsense firewalls | FreeBSD | ❌ | ❌ (no git) | SSH |
| CI runners | Linux | ✅ | ✅ (CI service account) | SSH |
| Dev/ops workstations (future) | Windows 11 / Server | ✅ | ✅ | **WinRM** |

**Cross-OS note:** Git behavior is identical across Linux and Windows. Only the **Ansible transport** differs — Linux hosts use SSH, Windows hosts use WinRM. The same `roles/git-config` role applies both layers via OS-specific task files (`tasks/linux.yml`, `tasks/windows.yml`). Paths differ (`/etc/gitconfig` vs `%ProgramData%\Git\config`) but property names and values are identical.

Windows support is **deferred** — current Ansible only targets Linux. When Win11 / Windows Server hosts are introduced, the role gains a Windows branch without changing this ADR's property table.

## Consequences

- All hosts with git get consistent config via Ansible — no per-host drift
- All commits and tags are signed. Unsigned commits are non-compliant.
- LFS is required; repos using LFS won't silently skip large files
- Credential helper caches HTTPS tokens for 1h — no repeated prompts
- `user.useConfigOnly=true` prevents silently-authored commits
- Git-host migration (GitHub → GitLab CE) requires no config change — the standard is host-agnostic

## CISO mapping

| Framework | Controls covered |
|---|---|
| ISO 27001:2022 | A.8.4, A.8.9, A.8.25 |
