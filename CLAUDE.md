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

## Naming Conventions

This repo follows the ORG naming convention. **Always check `docs/naming-convention.md` before creating new files.**

- **Repo pattern:** `{scope}-{component}-{qualifier}` — all lowercase, hyphens only
- **Branch pattern:** `docs/{issue-id}-{short-description}` (e.g. `docs/12-add-vault-adr`)
- **Commit pattern:** `{type}({scope}): {description}` (Conventional Commits)
- **ADR files:** `docs/adr/{NNNN}-{kebab-title}.md`

Full reference: [`docs/naming-convention.md`](docs/naming-convention.md)

---

## Commit Types (Docs Repo Only)

Only these commit types are used in this repo:

| Type | Use |
|---|---|
| `docs` | Documentation content changes |
| `chore` | Maintenance (rename, reorganise, update template) |
| `fix` | Correct errors, broken links, or outdated info |

Do **not** use `feat`, `ci`, `build`, `refactor`, or other code-oriented types in this repo.

---

## Branch Naming

```
docs/{issue-id}-{subject}
```

Examples:
- `docs/7-add-networking-adr`
- `docs/15-update-stack-references`
- `docs/42-document-secret-rotation`

---

## Agent Instructions

1. **Always read `docs/naming-convention.md` before creating any new files.** File names, paths, and structures must follow ORG conventions.
2. **Always read `docs/stack.md`** when writing about platform components, tools, or services — use the canonical names.
3. This is a documentation repo. Stay in your lane: edit `.md` files only.
4. ADRs go in `docs/adr/` with sequential numbering (`0001-`, `0002-`, etc.).
5. Use present tense in documentation. Keep it concise and factual.
6. When in doubt about naming: check `docs/naming-convention.md`. Don't guess.

---

## Links

- GitHub repo: `https://github.com/by-openclaw/doc-platform-core`
- Org: `https://github.com/by-openclaw`

---

## Agent Onboarding (Rune / BY-SYSTEMS)

This repo is part of the BY-SYSTEMS PoC platform maintained by Rune.

- **AGENTS.md:** [`AGENTS.md`](AGENTS.md) — generic agent onboarding file (read by Codex, Claude Code, and all agents)
- **Owner:** @yboujraf
- **Org:** [by-openclaw](https://github.com/by-openclaw)
- **Platform agent:** Rune (DevOps familiar)

### Quick Checklist for Any Agent

- [ ] Read `docs/naming-convention.md` before creating any file
- [ ] Read `docs/stack.md` before writing about any platform component
- [ ] Markdown only — no code, no scripts, no CI
- [ ] Conventional Commits with docs-only types (`docs`, `chore`, `fix`)
- [ ] ADRs go in `docs/adr/` with next sequential number
