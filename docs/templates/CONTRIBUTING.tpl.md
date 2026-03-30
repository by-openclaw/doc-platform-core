# Contributing to {{REPO_NAME}}

> Read [README.md](README.md) and [CLAUDE.md](CLAUDE.md) before starting.

---

## Before You Start

- Check existing issues and [RAID.md](RAID.md) for known risks
- Open or link an issue before significant work
- For architecture-impacting changes, create or update an ADR in `docs/adr/`

---

## Setup

<!-- Repo-specific setup instructions -->
```bash
# TODO: Add setup commands
```

---

## Branch Naming

```text
{type}/{issue-id}-{short-description}
```

Types: `feat`, `fix`, `docs`, `chore`, `refactor`, `test`, `sec`

---

## Commit Standard

Conventional Commits — `type(scope): description`

```text
feat({{SCOPE}}): add new capability
fix({{SCOPE}}): correct bug
docs({{SCOPE}}): update documentation
```

| Type | Version bump | When |
|---|---|---|
| `fix:` | patch | Bug fixes |
| `feat:` | minor | New features |
| `feat!:` / `BREAKING CHANGE:` | major | Breaking changes |
| `docs:` `test:` `chore:` | none | Non-functional |

---

## Definition of Done — PR Checklist

<!-- Adapt per repo — remove irrelevant items, add repo-specific ones -->
- [ ] Linting passes
- [ ] Tests pass (if applicable)
- [ ] No secrets in committed files
- [ ] Naming convention followed
- [ ] CHANGELOG.md entry added
- [ ] CLAUDE.md current state updated if applicable
- [ ] RAID.md + GitHub Issue + Projects board (if new risk/issue found)

---

## Diagram Standard

- Source: PlantUML `.puml` → `assets/diagrams/`
- Render: PNG via Kroki → `assets/exports/`
- Docs link to `assets/exports/` only

---

## Post-Release Checklist

After each release:
- [ ] AGENTS.md — update Project Stats
- [ ] CLAUDE.md — update Current State if changed
- [ ] README.md — update version

Commit: `docs: update project docs to v{version}`

---

## Release Process

- Use conventional commits consistently
- Release Please determines version bumps automatically
- Never manually edit version strings

---

## Archive Policy

Historical docs move to `docs/archive/` — never delete.
