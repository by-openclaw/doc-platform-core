> **Mandatory — read before any work:**
> 1. `workspace/OPERATING-STANDARD.md` — platform rules, quality gates, compliance
> 2. This file — repo-specific context

# CLAUDE.md — AI Agent Context for `doc-platform-core`

> **Scope:** `doc` | **Component:** `platform`
> **GitHub path:** `by-openclaw/doc-platform-core`

This file is read by AI agents (Claude Code, Codex, etc.) for context before working in this repository. Keep it up to date.

---

## What This Repo Does

`doc-platform-core` is the central documentation repository for the ORG platform. It contains architecture decisions, stack references, naming conventions, and platform-wide documentation. This repo contains **no code** — its sole purpose is to maintain accurate, well-structured platform documentation for engineers, operators, and agents working within the ORG ecosystem.

---

## Key Files — Always Read First

Before creating or editing any files, agents **must** read:

- [`docs/stack.md`](docs/stack.md) — full platform stack reference
- [`docs/naming-convention.md`](docs/naming-convention.md) — all naming patterns (files, branches, repos, services)

---

## Repository Layout

```
.
├── assets/                # Diagrams, images, and static assets
├── docs/
│   ├── adr/               # Architecture Decision Records (0001-*.md)
│   ├── archive/           # Archived session notes and superseded docs
│   ├── templates/         # Document templates (*.tpl.md)
│   ├── architecture.md    # Platform architecture overview
│   ├── idempotency-strategy.md  # Idempotency patterns and free plan notes
│   ├── naming-convention.md  # Naming standards (READ BEFORE CREATING FILES)
│   ├── netbox.md          # NetBox deep-dive (CMDB/IPAM/DCIM source of truth)
│   ├── stack.md           # Full platform stack reference
│   ├── roadmap.md         # Platform roadmap
│   └── raid.md            # Risks, Assumptions, Issues, Dependencies
└── README.md
```

---

## This Is a Docs-Only Repo

| What | Status |
|---|---|
| Build | ❌ None |
| Tests | ❌ None |
| Deploy | ❌ None |
| CI/CD pipeline | ❌ None |
| Code | ❌ None |

Do **not** add source code, Dockerfiles, CI pipelines, or build tooling to this repo. If you find yourself writing code, you're in the wrong repo.

---

## Naming, Commit, and Branch Conventions

> Commit types, branch naming, naming conventions, and agent onboarding → see [`AGENTS.md`](AGENTS.md).
> Full naming reference: [`docs/naming-convention.md`](docs/naming-convention.md).

---

## Cross-repo References

- Naming convention: see `docs/adr/0010-naming-and-identity-convention.md`
- Environment tiers: poc/dev/test/staging/acc/prod — always explicit. See `docs/adr/0012-environment-tier-standard.md`

---

## Links

- GitHub repo: `https://github.com/by-openclaw/doc-platform-core`
- Org: `https://github.com/by-openclaw`

---

## GitHub → Discord Release Webhook
This repo has a GitHub webhook configured for `release` events → Discord `#releases` channel (by-openclaw standard).
No discord-notify.yml workflow. No DISCORD_WEBHOOK secret. Discord-native parsing.
See `workspace/docs/stack.md` for the full standard and command to replicate on new repos.
