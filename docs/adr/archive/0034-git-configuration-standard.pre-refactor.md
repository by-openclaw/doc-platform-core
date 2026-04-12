# ADR-0034: Git Configuration Standard

**Status:** Accepted
**Date:** 2026-04-10
**Deciders:** @yboujraf

## Context

Git configuration is inconsistent across VMs. Some have signing enabled, others don't. Line endings, default branches, and credential helpers vary. The odoo-install/ssh/git.sh script sets many properties but misses cross-platform essentials (autocrlf, fetch.prune, credential.helper). A standard config is needed for all VMs that use git.

## Decision

### 1. Two-Layer Configuration

| Layer | File | Scope | Managed by | Content |
|---|---|---|---|---|
| System | `/etc/gitconfig` | All users on VM | Ansible `roles/git-config` | Core, protocol, fetch, transfer, LFS, credential |
| User | `~/.gitconfig` | Per user | Ansible `roles/git-config` | Identity, signing, GPG |

### 2. System-Wide Properties (`/etc/gitconfig`)

| Property | Value | Priority | Reason |
|---|---|---|---|
| `core.autocrlf` | `input` | MUST | LF in repo, native on checkout |
| `core.eol` | `lf` | MUST | Force LF |
| `core.filemode` | `true` | MUST | Respect permissions |
| `core.quotePath` | `false` | NICE | Show Unicode paths |
| `init.defaultBranch` | `main` | MUST | ADR-0019 |
| `color.ui` | `auto` | MUST | Colored output |
| `protocol.version` | `2` | MUST | Git protocol v2 |
| `fetch.prune` | `true` | MUST | Clean stale remote branches |
| `fetch.fsckObjects` | `true` | MUST | Validate on fetch |
| `transfer.fsckObjects` | `true` | MUST | Validate on transfer |
| `http.sslVerify` | `true` | MUST | Never disable SSL |
| `pull.rebase` | `true` | MUST | Clean history |
| `push.default` | `current` | NICE | Push current branch |
| `push.autoSetupRemote` | `true` | NICE | Auto-track on first push |
| `push.followTags` | `true` | NICE | Tags follow commits |
| `merge.ff` | `false` | NICE | No fast-forward (ADR-0019 --no-ff) |
| `merge.conflictStyle` | `zdiff3` | NICE | Better conflict markers |
| `diff.algorithm` | `histogram` | NICE | Better diffs |
| `diff.colorMoved` | `zebra` | NICE | Highlight moved code |
| `log.date` | `iso` | NICE | ISO date format |
| `log.abbrevCommit` | `true` | NICE | Shorter hashes |
| `credential.helper` | `cache --timeout=3600` | MUST | HTTPS token cache 1h |
| `filter.lfs.clean` | `git-lfs clean -- %f` | MUST | LFS support |
| `filter.lfs.smudge` | `git-lfs smudge -- %f` | MUST | LFS support |
| `filter.lfs.process` | `git-lfs filter-process` | MUST | LFS support |
| `filter.lfs.required` | `true` | MUST | Fail if LFS not installed |

### 3. Per-User Properties (`~/.gitconfig`)

| Property | Value | Priority | Reason |
|---|---|---|---|
| `user.name` | `svc-rune` or `by-systems` | MUST | Commit author |
| `user.email` | `svc-rune@by-systems.be` | MUST | Commit email |
| `user.signingKey` | GPG fingerprint | MUST | Commit signing key |
| `user.useConfigOnly` | `true` | MUST | Prevent guessing |
| `commit.gpgSign` | `true` | MUST | Sign all commits |
| `tag.gpgSign` | `true` | MUST | Sign all tags |
| `tag.forceSignAnnotated` | `true` | NICE | Force sign annotated |
| `gpg.program` | `gpg` | MUST | GPG binary |
| `gpg.format` | `openpgp` | MUST | Not x509 |

### 4. Jinja2 Template — System (`/etc/gitconfig`)

```jinja2
# {{ ansible_managed }}
# ADR-0034: Git Configuration Standard
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

### 5. Jinja2 Template — Per-User (`~/.gitconfig`)

```jinja2
# {{ ansible_managed }}
# ADR-0034: Git Configuration Standard — user identity
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

### 6. What NOT to Configure

| Property | Reason |
|---|---|
| `core.editor` | User preference, not standardized |
| `alias.*` | User preference |
| `pager.*` | User preference |
| `ssh.variant` | Auto-detected |
| `http.proxy` | Environment-specific, not global |

### 7. Applicability

| VM/LXC type | System gitconfig | Per-user gitconfig |
|---|---|---|
| Dev/ops VMs (vm-rune, vm-terraform) | YES | YES |
| Service LXCs (webdmz, websrv) | YES | NO (no git operations) |
| OPNsense FW | NO | NO (no git) |
| CI runners | YES | YES (CI service account) |

## Consequences

- All VMs with git get consistent configuration via Ansible
- Commits are always signed — unsigned commits are non-compliant
- LFS is required — repos using LFS won't silently skip large files
- fetch.prune keeps local branches clean automatically
- Credential helper caches HTTPS tokens for 1 hour — no repeated prompts
- Per-user identity prevents "guessed" author commits

## CISO mapping

### ISO/IEC 27001:2022

| Control | Title | Status | Notes |
|---|---|---|---|
| A.8.4 | Access to source code | Covered | Signed commits trace authorship |
| A.8.25 | Secure development lifecycle | Covered | Consistent git config, signing enforced |

## References

- ADR-0019: Git Workflow & Approval Process
- ADR-0033: User Identity & Access Standard
- Audit source: by-systems/odoo-install/ssh/git.sh
