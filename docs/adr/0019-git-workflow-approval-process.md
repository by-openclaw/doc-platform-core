# ADR-0019: Git Workflow and Approval Process

- **Status:** Accepted
- **Date:** 2026-04-02
- **Deciders:** @yboujraf

## Context

Multiple agents (Rune, Opus) and human contributors interact with the same repositories. Without a defined workflow, agents risk committing directly to main, force-pushing, or bypassing review. A lightweight but explicit git workflow is needed that fits the current small-team reality while enforcing Conventional Commits and protecting the main branch.

## Decision

**VCS:** GitHub now. Migration to GitLab CE self-hosted when GitLab is deployed (see ADR-0005). This ADR governs both phases — the workflow is the same regardless of host.

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

**Agent boundary:** Agents (Rune, Opus, sub-agents) may commit to feature branches and open PRs. Agents may not merge PRs to `main` or push directly to `main`. @yboujraf is the sole merge authority.

## Consequences

- All agent work on `main` goes through a PR. No exceptions.
- Force-push protection means rebasing against main requires `--force-with-lease` on feature branches only.
- Conventional Commits requirement means all commit messages must be machine-parseable. This enables automated CHANGELOG generation.
- Migration from GitHub to GitLab CE will require re-applying branch protection rules and PR templates to the new host.
- The tagging convention (`YYYY.M.D` for infra) allows point-in-time rollback of infrastructure state without coupling to semver.

## References

- ADR-0005 — VCS & CI/CD Strategy
- ADR-0006 §4 — Branch strategy and contribution flow
- `brainstorming/2026-04-01-git-flow-approval-process.md` — source brainstorming doc
