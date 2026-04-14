# CLAUDE.md — AI Agent Context for `doc-platform-core`

> **Mandatory — read before any work:**
> 1. [`OPERATING-STANDARD.md`](OPERATING-STANDARD.md) — platform rules, quality gates, session protocol (applies to every repo, every session)
> 2. This file — repo-specific context

**Scope:** `doc` | **Component:** `platform`
**GitHub path:** `by-openclaw/doc-platform-core`

This file is read by AI agents (Claude Code, Codex, OpenAI, etc.) for context before working in this repository. Keep it up to date.

---

## What this repo does

`doc-platform-core` is the **central platform documentation repository** for BY-SYSTEMS. It holds:

- **Scoped Architecture Decision Records** (30 ADRs across 7 scopes — identity, git, naming, security, infra, services, lib/python)
- **Operational rules** that apply to every BY-SYSTEMS repo (`OPERATING-STANDARD.md`)
- **Live platform state** (`docs/status.md`, `docs/roadmap.md`, `docs/raid.md`)
- **Document templates** used when bootstrapping new repos (`docs/templates/`)
- **Archives** of pre-refactor decisions (read-only, historical reference only)

This repo contains **no code** — its sole purpose is accurate, well-structured platform documentation for engineers, operators, and AI agents working within the BY-SYSTEMS ecosystem.

---

## Key files — always read first

Before creating or editing any files, agents **must** read in this order:

1. [`OPERATING-STANDARD.md`](OPERATING-STANDARD.md) — platform rules
2. [`docs/adr/README.md`](docs/adr/README.md) — scoped ADR index (30 ADRs across 7 scopes)
3. [`docs/adr/naming/0001-infra.md`](docs/adr/naming/0001-infra.md) — **mandatory before creating any infrastructure-related file** (hostnames, FQDNs, DNS, VMs, LXCs)
4. [`docs/adr/naming/0004-automation.md`](docs/adr/naming/0004-automation.md) — **mandatory before creating any repo, Python, Ansible, or Terraform artifact**

---

## Repository layout

```
.
├── OPERATING-STANDARD.md   # Platform rules — mandatory
├── README.md               # Repo overview
├── CLAUDE.md               # This file
├── AGENTS.md               # Generic agent onboarding
├── CONTRIBUTING.md         # Contribution guide
├── SECURITY.md             # Vulnerability disclosure
├── RAID.md                 # Repo-scoped RAID log
├── CHANGELOG.md            # Auto-generated (Release Please)
├── LICENSE                 # CC BY-SA 4.0
│
├── docs/
│   ├── adr/
│   │   ├── README.md                # Scoped ADR index
│   │   ├── identity/                # 4 ADRs (authentication, provisioning, machine creds, OS accounts)
│   │   ├── git/                     # 3 ADRs (workflow, platform strategy, configuration)
│   │   ├── naming/                  # 4 ADRs (infra, identity, firewall, automation)
│   │   ├── security/                # 5 ADRs (secret storage, compliance, hardening, certs, licensing)
│   │   ├── infra/                   # 8 ADRs (stack, charter, terraform, network, env tiers, logging, monitoring, backup)
│   │   ├── services/                # 5 ADRs (opnsense, email, netbox CMDB, database, notifications)
│   │   ├── lib/python/              # 1 ADR (Python library design standard)
│   │   └── archive/                 # Pre-refactor flat ADR snapshots — read-only
│   │
│   ├── status.md                    # Live platform state
│   ├── roadmap.md                   # Forward-looking roadmap
│   ├── raid.md                      # Platform-wide RAID log (per OS §7.1)
│   ├── templates/                   # Document templates
│   └── archive/2026-04-14-pre-cleanup/   # Pre-cleanup snapshots of loose files — read-only
│
└── assets/                 # Diagrams, images, static assets
```

---

## This is a docs-only repo

| What | Status |
|---|---|
| Source code | ❌ None |
| Dockerfiles | ❌ None |
| CI/CD pipelines | ❌ None — only `release-please.yml` + `project-board-sync.yml` |
| Build tooling | ❌ None |
| Tests | ❌ None |

Do **not** add source code, Dockerfiles, CI pipelines, or build tooling to this repo. If you find yourself writing code, you're in the wrong repo.

---

## Naming, commit, and branch conventions

- **Commit types used in this repo:** `docs`, `chore`, `fix` only. **Do not use** `feat`, `ci`, `build`, `refactor`, `test` — see `CONTRIBUTING.md`.
- **Branch naming:** `{type}/{issue-id}-{short-description}` per `OPERATING-STANDARD.md §4.3`
- **ADR path:** `docs/adr/{scope}/NNNN-short-title.md` (scoped folders, per-scope numbering) — see `docs/adr/README.md`
- **Full naming reference:** `docs/adr/naming/0001-infra.md` (hosts, FQDNs, DNS), `docs/adr/naming/0004-automation.md` (repos, Python, Ansible, Terraform, tests)

---

## Cross-references to authoritative decisions

When a decision spans multiple repos, this repo is the source. Key cross-references agents should follow:

| Concern | Authoritative ADR |
|---|---|
| Hostname / FQDN / DNS / NetBox-as-source-of-truth | `docs/adr/naming/0001-infra.md` |
| Account and group naming | `docs/adr/naming/0002-identity.md` |
| OPNsense firewall alias and rule naming | `docs/adr/naming/0003-firewall.md` |
| Repo / Python / Ansible / Terraform / test naming | `docs/adr/naming/0004-automation.md` |
| Environment tiers (`dev/test/staging/acc/prod/drp`, 6 tiers, **no `poc`**) | `docs/adr/infra/0005-environment-tiers.md` |
| Network segments and VLAN registry | `docs/adr/infra/0004-network-architecture.md` |
| Git workflow (branching, commits, merge, agent boundary) | `docs/adr/git/0001-workflow.md` |
| Authentication (Authentik as IAM hub) | `docs/adr/identity/0001-authentication.md` |
| OS accounts, SSH/GPG keys, sudo, break-glass | `docs/adr/identity/0004-os-accounts.md` |
| Secret storage (HashiCorp Vault + Vaultwarden) | `docs/adr/security/0001-secret-storage.md` |
| Certificate strategy (Let's Encrypt + step-ca) | `docs/adr/security/0004-certificate-strategy.md` |
| Licensing policy (approved / flagged SPDX lists) | `docs/adr/security/0005-licensing-policy.md` |
| Python library design standard | `docs/adr/lib/python/0001-design-standard.md` |

Never reference flat ADR numbers (`ADR-0001`, `ADR-0010`, etc.) — they are archived. Always reference the scoped path.

---

## Agent identity and human profile — not in this repo

Per `OPERATING-STANDARD.md §3.2 File ownership`, `SOUL.md` and `USER.md` are **workspace-level** files. They live at `~/.openclaw/workspace/SOUL.md` and `~/.openclaw/workspace/USER.md`, not in this repo. Agents reading this repo find agent identity and owner profile in the workspace.

---

## Links

- GitHub repo: https://github.com/by-openclaw/doc-platform-core
- Org: https://github.com/by-openclaw
- Owner: @yboujraf
- Agent: Rune — DevOps familiar for BY-SYSTEMS

---

## GitHub → Discord release webhook

This repo has a GitHub webhook configured for `release` events → Discord `#releases` channel (by-openclaw standard). **No `discord-notify.yml` workflow. No `DISCORD_WEBHOOK` secret.** Discord-native parsing.

See `services/0005-notifications` and the webhook configuration in the BY-SYSTEMS platform runbook for how to replicate on new repos.
