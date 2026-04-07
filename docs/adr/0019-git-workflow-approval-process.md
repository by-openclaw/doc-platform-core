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

## PR Content Standard

> Added 2026-04-06 — all PRs must include structured review context.

Every PR description must include the following sections. Sections marked (if applicable) may be omitted when not relevant to the change.

### 1. Files changed table (mandatory)

| File | Type | Change |
|------|------|--------|
| `path/to/file.py` | new / fix / update / delete | One-line description |

### 2. Test results table (if code changed)

| Suite | Scope | File | Passed | Failed |
|-------|-------|------|--------|--------|
| Unit | ManagerName | `tests/unit/test_manager.py` | 16 | 0 |
| Integration | Domain | `tests/integration/test_lifecycle.py` | 14 | 0 |
| **Full suite** | **All** | `tests/` | **226** | **0** |
| Lint | ruff / ansible-lint / terraform fmt | `src/` + `tests/` | clean | — |

### 3. Endpoints / resources covered (if API/infra code)

For `lib-*` repos:

| Endpoint | Method | Tested |
|----------|--------|--------|
| `/api/domain/action` | POST | unit + integration |

For `infra-terraform-*` repos:

| Module / Resource | Action | Tested |
|------------------|--------|--------|
| `module/vm-opnsense` | plan + apply | yes |

### 4. Safety boundaries (if touching live systems)

- **READ-ONLY:** what must NOT be modified on the live device
- **CRUD safe:** what can be created/deleted with `inttest-` prefix
- **DISABLED only:** what must be created in disabled state

### 5. How to review (mandatory)

Numbered steps guiding the reviewer through the change:
1. Read X — verify Y
2. Check Z — confirm W

### Commit message format

One commit per concern. Structure:

```
type(scope): title

Managers/modules/resources:
  - Name: endpoint/path
    File: path/to/file.py

Tests:
  - Name unit (N tests): path/to/test.py
  - Name integration (N tests): path/to/test.py

Fixes (if any):
  - Description
    File: path/to/file.py

Files changed:
  path/to/file1.py  (new/fix/update)
  path/to/file2.py  (new/fix/update)

Co-Authored-By: ...
```

## Consequences

- All agent work on `main` goes through a PR. No exceptions.
- Force-push protection means rebasing against main requires `--force-with-lease` on feature branches only.
- Conventional Commits requirement means all commit messages must be machine-parseable. This enables automated CHANGELOG generation.
- Migration from GitHub to GitLab CE will require re-applying branch protection rules and PR templates to the new host.
- The tagging convention (`YYYY.M.D` for infra) allows point-in-time rollback of infrastructure state without coupling to semver.

## References

- VCS & CI/CD strategy decision defines the GitLab CE migration target and rationale
- Platform charter defines branch strategy and contribution flow principles
- `brainstorming/2026-04-01-git-flow-approval-process.md` — source brainstorming doc
