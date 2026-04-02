# AGENTS.md — doc-platform-core

Central documentation repository for the BY-SYSTEMS platform — architecture, ADRs, stack reference, RAID, and naming conventions. **No code lives here.**

## Always Read First

Before creating or editing anything in this repo:

1. [`README.md`](README.md) — repo structure and overview
2. [`CLAUDE.md`](CLAUDE.md) — agent-specific constraints and instructions
3. [`docs/naming-convention.md`](docs/naming-convention.md) — **mandatory before creating any file**
4. [`docs/stack.md`](docs/stack.md) — canonical platform stack reference
5. [`docs/raid.md`](docs/raid.md) — risks, assumptions, issues, dependencies

## Mandatory reading before acting

Before writing, editing, or reviewing any file in this repo, read:

### doc-platform-core repo:
1. `/home/by-systems/.openclaw/workspace/repos/doc-platform-core/docs/standards/` — all standards files
2. `/home/by-systems/.openclaw/workspace/repos/doc-platform-core/docs/adr/` — all Accepted ADRs
3. The ADR template for your scope: `doc-platform-core/docs/templates/adr-template-infra.md`

### lib repos (lib-synology-dsm etc.):
1. `/home/by-systems/.openclaw/workspace/repos/lib-synology-dsm/docs/adr/` — lib-scoped ADRs only
2. The ADR template for your scope: `doc-platform-core/docs/templates/adr-template-lib.md`
3. Platform standards are NOT binding on lib repos — but lib CISO sections must reference them

### Rules:
- Do NOT infer. Do NOT invent policy. If a standard or ADR covers it — follow it.
- If you would override a standard — flag it with `[OVERRIDE REQUIRED]`, do NOT do it silently.
- Cross-ADR dependencies are FORBIDDEN. Each ADR is self-contained. Do not say "see ADR-XXXX".
- If content is relevant to two ADRs — each states it independently within its own scope.

## Coding & Commit Standards

- **Conventional Commits** — docs repo uses limited types only:
  - `docs(scope): description` — content changes
  - `chore(scope): description` — maintenance, rename, reorganise
  - `fix(scope): description` — correct errors, broken links, outdated info
  - ❌ Do NOT use `feat`, `ci`, `build`, `refactor` in this repo
- **Branch naming:** `docs/{issue-id}-{subject}`
  - Example: `docs/7-add-networking-adr`
- **ADR numbering:** sequential, zero-padded — `docs/adr/NNNN-kebab-title.md`
- **File names:** all lowercase, hyphens only, `.md` extension

## What NOT To Do

> Also read [`CLAUDE.md`](CLAUDE.md) for repo layout, docs-only constraints, and key file references.

- ❌ Do NOT add source code, scripts, Dockerfiles, or CI pipelines
- ❌ Do NOT create files without checking naming conventions first
- ❌ Do NOT use `feat` or code-oriented commit types
- ❌ Do NOT add binary assets without Git LFS configured (`.gitattributes`)
- ❌ Do NOT skip ADR sequence numbers — check the highest existing ADR first
- ❌ Do NOT write in future tense — use present tense in all documentation

## GitHub Repo

<https://github.com/by-openclaw/doc-platform-core>

## Agent: Rune

Maintained by Rune (DevOps familiar) for the BY-SYSTEMS PoC platform.
Owner: @yboujraf

## Doc Maintenance — After Every Successful Build

After each successful CI build (all jobs green), update these files to reflect current state:
- **AGENTS.md** — Update "Project Stats", version, checklist, roadmap progress
- **CLAUDE.md** — Update build commands, file table, current state if anything changed
- **README.md** — Update badges, feature lists, version numbers

Commit separately: `docs: update project docs to v{version}`

This ensures any AI agent (or human) picking up the project always has accurate, current documentation.

---

## Project Stats

> Auto-updated on every release. Last updated: 2026-03-31

| Metric | Value |
|---|---|
| Version | v0.5.1 |
| Tagged releases | 3 |
| Total files | 64 |
| Documentation files | 37 |
| Python source files | 0 |
| Test files | 0 |
| Terraform files | 0 |
| YAML/Ansible files | 1 |
| ADR decisions | 13 |
| CI workflows | 1 |

