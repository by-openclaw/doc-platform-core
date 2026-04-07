# ADR-0019: Git Workflow and Approval Process

- **Status:** Accepted
- **Date:** 2026-04-02
- **Deciders:** @yboujraf

## Context

Multiple agents (Rune, Opus) and human contributors interact with the same repositories. Without a defined workflow, agents risk committing directly to main, force-pushing, or bypassing review. A lightweight but explicit git workflow is needed that fits the current small-team reality while enforcing Conventional Commits and protecting the main branch.

## Decision

**VCS:** GitHub now. Migration to GitLab CE self-hosted when GitLab is deployed. This ADR governs both phases — the workflow is the same regardless of host.

**Branching model:** Trunk-based development. `main` is the single long-lived branch. No `develop`, no `release/*` branches in PoC. Feature branches are short-lived (PR → merge → delete).

**Commit format:** Conventional Commits enforced. Format: `type(scope): description`. Types: `feat`, `fix`, `docs`, `chore`, `refactor`, `test`, `ci`. Pre-commit hook or CI gate enforces this.

**Protected branch rules for `main`:**
- No direct push to `main` (agents or humans)
- No force-push to `main` — ever
- PRs require at least one human approval before merge
- Agents open PRs, never merge them

**Tagging:**
- Software releases: semver (`v1.2.3`)
- Infrastructure snapshots: `YYYY.M.D` (e.g. `2026.4.2`)

**Merge strategy:** Rebase + merge commit (`--no-ff`), following Odoo/OCA workflow.

### Contributor flow (before requesting merge)

```bash
# 1. Create feature branch from main
git checkout -b feat/xxx main

# 2. Work, commit (Conventional Commits, one logical change per commit)
git commit -m "feat(scope): description"
git commit -m "fix(scope): description"

# 3. Before push: rebase on latest main
git fetch origin main
git rebase origin/main

# 4. During rebase: squash fixup/WIP commits (keep meaningful commits)
git rebase -i origin/main
#   pick  abc1234 feat(scope): add manager
#   fixup def5678 WIP: fix typo        ← squash noise
#   pick  ghi9012 test(scope): add unit tests  ← keep

# 5. Push (first time or after rebase)
git push -u origin feat/xxx
# or after rebase of already-pushed branch:
git push --force-with-lease
```

### Merge authority flow (@yboujraf)

```bash
# 6. Review PR on GitHub — verify:
#    - Commits are clean (no WIP/fixup)
#    - Branch is rebased on main (no merge conflicts)
#    - CI green

# 7. Merge via GitHub "Merge commit" button (--no-ff)
#    This creates a merge commit = PR audit trail
```

### Result

- Every meaningful commit stays in history (no squash of logical changes)
- Merge commit marks the PR boundary (auditable)
- Linear history (rebase) + visible PR grouping (merge commit)

| Scenario | Contributor action | Merge action |
|---|---|---|
| Clean PR (each commit = one logical change) | `git rebase origin/main` | Merge commit (`--no-ff`) |
| PR with fixup/WIP commits | `git rebase -i` to squash fixups | Merge commit (`--no-ff`) |
| Single-commit PR | `git rebase origin/main` | Merge commit (`--no-ff`) |

### GitHub repo settings (all repos)

| Setting | Value | Why |
|---|---|---|
| Allow merge commits | **Enabled** | `--no-ff` merge = PR audit trail |
| Allow squash merging | **Disabled** | Destroys commit history |
| Allow rebase merging | **Disabled** | Loses PR boundary (no merge commit) |

**Rationale:** Matches Odoo/OCA industry standard. Linear commit history (rebase by contributor) + visible PR boundaries (merge commit by authority) + full audit trail.

**Agent boundary:** Agents (Rune, Opus, sub-agents) may commit to feature branches and open PRs. Agents may not merge PRs to `main` or push directly to `main`. @yboujraf is the sole merge authority.

## CISO mapping

> Applies only to controls directly relevant to this ADR's scope.

### ISO/IEC 27001:2022

| Control | Title | Status | Notes |
|---|---|---|---|
| A.8.32 | Change management | ✓ Covered | PR review with human approval required; no direct push to main |
| A.8.9 | Configuration management | ✓ Covered | All changes versioned; infrastructure snapshots tagged by date |
| A.5.12 | Classification of information | ✓ Covered | Conventional Commits enables automated CHANGELOG and release traceability |

### NIS2 (Directive 2022/2555)

| Article | Requirement | Status | Notes |
|---|---|---|---|
| Art. 21(2)(e) | Security in network and information systems acquisition | ✓ Covered | Change management via PR; @yboujraf is sole merge authority |

### GDPR (Regulation 2016/679)

Not applicable — this ADR covers git workflow, not personal data processing.

## Consequences

- All agent work on `main` goes through a PR. No exceptions.
- Force-push protection means rebasing against main requires `--force-with-lease` on feature branches only.
- Merge strategy (`--no-ff` after rebase) preserves full commit history with PR boundaries. Contributors must rebase before merge — the merge authority (@yboujraf) verifies clean history before merging.
- Conventional Commits requirement means all commit messages must be machine-parseable. This enables automated CHANGELOG generation.
- Migration from GitHub to GitLab CE will require re-applying branch protection rules and PR templates to the new host.
- The tagging convention (`YYYY.M.D` for infra) allows point-in-time rollback of infrastructure state without coupling to semver.

## References

- VCS & CI/CD strategy decision defines the GitLab CE migration target and rationale
- Platform charter defines branch strategy and contribution flow principles
- `brainstorming/2026-04-01-git-flow-approval-process.md` — source brainstorming doc
