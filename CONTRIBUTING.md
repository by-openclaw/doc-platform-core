# Contributing to doc-platform-core

> Read [README.md](README.md) and [CLAUDE.md](CLAUDE.md) before starting.
> **This is a docs-only repo.** No code, no scripts, no CI pipelines.

---

## Before You Start

- Read `docs/naming-convention.md` — all file names must follow ORG conventions
- Read `docs/stack.md` — use canonical names for platform components
- Check `docs/raid.md` for open risks and issues

---

## Branch Naming

```text
docs/{issue-id}-{short-description}
```

Examples: `docs/7-add-networking-adr`, `docs/15-update-stack-references`

---

## Commit Types (docs repo only)

| Type | Use |
|---|---|
| `docs` | Documentation content changes |
| `chore` | Maintenance (rename, reorganise, update template) |
| `fix` | Correct errors, broken links, or outdated info |

**Do not use** `feat`, `ci`, `build`, `refactor` in this repo.

---

## ADR Process

Store ADRs in `docs/adr/` with sequential zero-padded numbering:

```text
docs/adr/0009-{kebab-title}.md
```

Create or update an ADR when changing:
- Platform architecture or topology
- Tool selection or replacement
- Security, identity, or networking strategy
- Naming or environment conventions

---

## File Naming

```text
{type}-{subject}-{YYYY-MM-DD}.md
```

Examples: `sow-infra-network-setup-2026-03-25.md`, `runbook-vault-backup-2026-03-25.md`

---

## Definition of Done — PR Checklist

- [ ] File naming follows `docs/naming-convention.md`
- [ ] Component names match `docs/stack.md` canonical names
- [ ] Markdown renders correctly (no broken links, images)
- [ ] ADR updated if architecture decision changed
- [ ] `docs/raid.md` updated if new risk/issue/dependency found
- [ ] CHANGELOG.md entry added

---

## Archive Policy

Historical docs move to `docs/archive/` — never delete. See lib-synology-dsm ADR-0004.

---

## Templates

Standardized templates for new repos:

| Template | Purpose |
|---|---|
| `docs/templates/CLAUDE.tpl.md` | Per-repo Claude contract |
| `docs/templates/CONTRIBUTING.tpl.md` | Contributing guidelines |
| `docs/templates/README.tpl.md` | README |
| `docs/templates/sow.tpl.md` | Statement of Work |

Update templates when repo-level improvements are proven (backport pattern).

---

## Release Process

- Use conventional commits consistently
- Release Please determines version bumps automatically
- Never manually edit version strings
