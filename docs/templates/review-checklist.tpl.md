# Repo Review Checklist — {repo-name}

> **Usage:** Run before declaring any repo done. Report pass/fail per item.
> **After running:** Commit results to `docs/audits/review-checklist-{repo}-{date}.md`

Set repo type before running:

```
TYPE=lib|infra|docs
```

---

## Files exist

- [ ] README.md — current, reflects actual state
- [ ] CLAUDE.md — hard rules, v1.0 blockers, current state updated
- [ ] AGENTS.md — stats current, reading order correct
- [ ] CONTRIBUTING.md — setup, commit, DoD, diagrams, post-release
- [ ] CHANGELOG.md — up to date
- [ ] SECURITY.md — supported version current
- [ ] RAID.md — repo-scoped risks, no stale items
- [ ] LICENSE — correct entity + author
- [ ] docs/adr/ — all decisions recorded, index updated
- [ ] docs/archive/ — directory exists
- [ ] .github/ISSUE_TEMPLATE/ — bug + feature (+ RAID if applicable)
- [ ] .github/PULL_REQUEST_TEMPLATE.md — checklist matches DoD
- [ ] .github/CODEOWNERS — owner set
- [ ] .github/workflows/release-please.yml — working
- [ ] .github/workflows/project-board-sync.yml — working

---

## Content aligned

- [ ] CLAUDE.md references OPERATING-STANDARD.md
- [ ] CLAUDE.md references naming convention ADR
- [ ] AGENTS.md and CLAUDE.md are not duplicated (dedup pattern applied)
- [ ] Secret references use `<REDACTED:{type}>` — no plaintext credentials
- [ ] Environment labels follow convention (poc/dev/test/staging/acc/prod)
- [ ] Service accounts follow naming (svc-{function}-{env})
- [ ] All ADRs have ## Compliance section with ISO/NIS2/GDPR tags

---

## Quality (repo-type dependent)

### Type: `lib` (lib-synology-dsm)

- [ ] CI green
- [ ] Tests pass
- [ ] Ruff clean
- [ ] Mypy clean
- [ ] Coverage at 100% (`# pragma: no cover` for conscious exceptions)

### Type: `infra` (infra-terraform-proxmox, ansible-platform, platform-setup)

- [ ] CI green (if CI exists)
- [ ] `terraform fmt` / `ansible-lint` clean (if applicable)
- [ ] No coverage or linting requirement — these are IaC/config repos

### Type: `docs` (doc-platform-core)

- [ ] Structure only — no CI quality gates
- [ ] Markdown renders correctly (no broken links)
- [ ] File naming follows `docs/naming-convention.md`

---

## Compliance

- [ ] No plaintext secrets in any committed file
- [ ] RAID.md reflects current risks
- [ ] ADRs tagged with compliance controls
- [ ] Account naming follows convention
