# git/0001 — Workflow & Approval Process

**Status:** Draft
**Date:** 2026-04-12 (supersedes flat ADR-0019, 2026-04-02)
**Scope:** How we branch, commit, review, merge, and tag — on any git host.
**Related:** `git/0002-platform-strategy`, `git/0003-configuration`, `identity/0003-machine-credentials`, `OPERATING-STANDARD.md §PR`

---

## Context

Multiple agents (Rune, Opus) and humans interact with the same repos. Without a defined workflow, agents risk committing directly to `main`, force-pushing, or bypassing review. This ADR defines the workflow — the PR content structure lives in `OPERATING-STANDARD.md`.

## Decision

**Branching:** Trunk-based. `main` is the single long-lived branch. Feature branches are short-lived (PR → merge → delete). No `develop`, no `release/*`.

**Commit format:** Conventional Commits, enforced: `type(scope): description`. Allowed types: `feat`, `fix`, `docs`, `chore`, `refactor`, `test`, `ci`, `security`.

**Merge strategy:** Contributor rebases on `main`, merge authority merges with `--no-ff` (merge commit). Linear history + visible PR boundaries. Matches Odoo/OCA industry standard.

**Tagging:**
- Software releases: semver (`v1.2.3`)
- Infrastructure snapshots: `YYYY.M.D` (e.g. `2026.4.2`)

## Protected `main` branch rules

- No direct push to `main`
- No force-push to `main` — ever
- PRs require at least one **human** approval before merge
- **Agents open PRs, never merge them**
- @yboujraf is the sole merge authority

## Contributor flow

```bash
# 1. Feature branch from main
git checkout -b feat/xxx main

# 2. Work, commit (one logical change per commit)
git commit -m "feat(scope): description"

# 3. Before push: rebase on main
git fetch origin main
git rebase origin/main

# 4. Squash fixup/WIP commits, keep meaningful ones
git rebase -i origin/main

# 5. Push (force-with-lease after rebase of pushed branch)
git push -u origin feat/xxx
git push --force-with-lease
```

## Merge authority flow

```bash
# Review PR — verify:
#   - Commits are clean (no WIP/fixup)
#   - Branch rebased on main
#   - CI green
#   - PR description follows OPERATING-STANDARD.md §PR

# Merge via GitHub/GitLab "Merge commit" button (--no-ff)
```

## Host repo settings (GitHub now, GitLab CE future)

| Setting | Value | Reason |
|---|---|---|
| Allow merge commits | **Enabled** | `--no-ff` merge = PR audit trail |
| Allow squash merging | **Disabled** | Destroys commit history |
| Allow rebase merging | **Disabled** | Loses PR boundary (no merge commit) |
| Require PR review | **1 human** | Agent approvals do not count |
| Force-push on `main` | **Forbidden** | No exceptions |

Branch protection is re-applied identically when the repo migrates from GitHub to GitLab CE (see `git/0002-platform-strategy`).

## Agent boundary

Agents (Rune, Opus, sub-agents) **may**:
- Commit to feature branches
- Open PRs
- Self-review for CI/lint

Agents **may not**:
- Push directly to `main`
- Merge PRs
- Force-push any branch without explicit per-action user approval

## PR content structure

Every PR description must follow `OPERATING-STANDARD.md §PR` — files-changed table, test results, endpoints, safety boundaries, how-to-review. Not duplicated here.

## Consequences

- All agent work reaches `main` through a PR — no exceptions
- Linear history + merge commits = auditable PR boundaries
- Rebasing pushed branches requires `--force-with-lease` (feature branches only)
- Conventional Commits enables automated CHANGELOG and release-please
- Host migration (GitHub → GitLab CE) = re-apply branch protection + PR template, no workflow change

## CISO mapping

| Framework | Controls covered |
|---|---|
| ISO 27001:2022 | A.5.12, A.8.9, A.8.32 |
| NIS2 | Art. 21(2)(e) |
