# Proposal: CI Pipeline, Workflows & Developer Platform Design

> **Scope:** All repos — platform-wide CI/workflow/quality standard
> **Reference implementation:** lib-synology-dsm (Python — apply here first, then templatize)
> **Date:** 2026-03-31
> **Author:** Claude Opus (audit session with @yboujraf)
> **Status:** APPROVED — ready for agent execution
> **Executed by:** Team agent — follows OPERATING-STANDARD.md
> **Audited by:** Claude Opus (after each phase, when asked by @yboujraf)
> **Company:** BY-SYSTEMS SRL
>
> **Context:** Derived from a full audit of lib-synology-dsm (18 audit files),
> cross-repo gap analysis (5 repos), and workspace-level review. lib-synology-dsm
> is the reference implementation. Once proven, patterns are extracted into
> `docs/templates/` for reuse on any new repo, any language.

---

## Scope

This proposal covers 12 topics to make lib-synology-dsm a production-grade, developer-friendly Python library. Each topic has: current state, what's proposed, and why.

**Rule:** Nothing is applied until you agree per topic.

---

## 1. CI Pipeline (`ci.yml`)

### Current state
- Matrix: Python 3.10/3.11/3.12/3.13 (good)
- Linting: ruff check + format (good)
- Type checking: mypy (good)
- Testing: pytest + coverage 80% floor (see topic 8 for threshold discussion)
- Security: bandit + pip-audit (good)
- **No build validation** — wheel/sdist never tested in CI

### Proposed changes

```
1.1  Add build validation job:
     - python -m build (wheel + sdist)
     - pip install dist/*.whl && python -c "import synology_dsm"
     - Ensures the package is installable from a clean build

1.2  Add nox as CI runner (replace raw pip install + pytest):
     - nox -s tests lint smoke
     - Single source of truth for local + CI test commands
     - Developers run the same nox sessions locally

1.3  Add SPDX license header check:
     - grep -rL "SPDX-License-Identifier: MIT" src/synology_dsm/*.py
     - Fails if any .py source file is missing the header
     - LICENSE author line: "Copyright (c) 2026 BY-SYSTEMS — Youssef Boujraf"

1.4  Add changelog validation:
     - Verify CHANGELOG.md has an [Unreleased] section or release entry
     - Prevents merges without changelog awareness

1.5  Pin CI action versions with SHA (security hardening):
     - actions/checkout@v4 → actions/checkout@<sha>
     - Prevents supply-chain attacks via tag mutation
```

### Diagram: CI pipeline flow

```
push to main/PR
  │
  ├─ lint job ──────── ruff check + ruff format + mypy + SPDX check
  │
  ├─ test job ─────── nox -s tests (3.10/3.11/3.12/3.13) + coverage
  │                    └─ artifacts: htmlcov + coverage.xml (30-day)
  │
  ├─ security job ─── bandit + pip-audit
  │
  ├─ build job ────── python -m build + pip install dist/*.whl
  │
  └─ (on main only)
     └─ release-please ── PR → merge → tag → GitHub Release
```

---

## 2. Release Please Workflow (`release-please.yml`)

### Current state
- Simple: runs on push to main, creates release PR
- No post-release steps

### Proposed changes

```
2.1  Add post-release notification step:
     - On release created → post summary to Discord #bot-openclaw
     - Use GitHub webhook (already configured) — no workflow changes needed
     - Verify webhook fires on release event (currently configured)

2.2  Add post-release doc sync reminder (non-blocking):
     - Comment on release PR: "Post-release: update AGENTS.md stats, CLAUDE.md state, README.md badges"
     - Implemented as a PR comment, not a blocking gate
```

---

## 3. Dev & Test Platform (nox)

### Current state
- `noxfile.py` exists with 4 sessions: tests, lint, smoke, integration
- CI doesn't use nox — runs raw pip install + pytest

### Proposed changes

```
3.1  CI uses nox as test runner:
     - Replace raw pip/pytest commands with nox -s tests lint smoke
     - Developers and CI run identical commands
     - Eliminates "works locally, fails in CI" drift

3.2  Add nox session: "build"
     - python -m build + install check
     - Matches CI build validation job (topic 1.1)

3.3  Add nox session: "spdx"
     - Check all .py files have SPDX-License-Identifier header
     - Matches CI SPDX check (topic 1.3)

3.4  Document nox in CONTRIBUTING.md (already done — verify):
     - nox (all sessions)
     - nox -s tests (test only)
     - nox -s lint (lint only)
     - nox -s smoke (import check, no network)
     - nox -s integration (requires NAS env vars)
```

---

## 4. Issue Templates

### Current state
- `bug_report.md` — DSM version, hardware, lib version, Python version (good)
- `feature_request.md` — DSM API, use case, proposed interface (good)
- Labels pre-filled: `bug`, `enhancement`

### Proposed changes

```
4.1  Add "chore" issue template:
     - For refactoring, dependency updates, CI changes
     - Labels: chore
     - Fields: what, why, scope

4.2  Add "security" issue template:
     - For security-related issues (non-vulnerability — those go to SECURITY.md)
     - Labels: security
     - Fields: component, risk, proposed fix

4.3  Add RAID issue templates (copy from platform-setup):
     - raid-risk.md, raid-action.md, raid-dependency.md
     - Labels: raid:risk, raid:action, raid:dependency
     - Ensures RAID atomic rule is followed for this repo too

4.4  Add issue form validation (YAML format):
     - Migrate from .md to .yml issue templates
     - Required fields enforced by GitHub (not optional)
     - Dropdown for DSM version, Python version
```

---

## 5. PEP Compliance

### Current state
- PEP 8: ruff enforced (good)
- PEP 257: partial (public = mandatory, private = optional)
- PEP 484/526: type hints on public API, mypy strict-ish
- PEP 517/518: pyproject.toml with hatchling (good)
- PEP 561: py.typed marker present (good)
- PEP 440: semver versioning (good)

### Proposed changes

```
5.1  Enable ruff D rules (pydocstyle) for public API:
     - Add "D" to ruff select for src/synology_dsm/
     - Ignore D1xx (missing docstrings on private methods)
     - Enforce D2xx/D3xx/D4xx (docstring formatting)

5.2  Add __all__ to all public modules:
     - Explicit public API surface per module
     - Helps IDE autocompletion + documentation generators

5.3  SPDX license identifier in all .py files (see topic 1.3):
     - # SPDX-License-Identifier: MIT
     - First line of every .py file in src/synology_dsm/
```

---

## 6. CONTRIBUTING.md enhancements

### Current state
- Setup (Path A + B), credentials, pre-commit, release, commit standard, DoD, diagrams, post-release, nox — all present

### Proposed changes

```
6.1  Add "How to add a new manager" section:
     - Step-by-step: create module, implement ensure(), add tests, update CLAUDE.md
     - Reference ADR-0002 (ensure pattern) and ADR-0007 (return dict)
     - Template for new manager class skeleton

6.2  Add "How to write an ADR" section:
     - When to write one (architecture-impacting decisions)
     - Copy 0000-template.md, fill in, submit PR
     - Reference OPERATING-STANDARD.md §9.2

6.3  Add "Troubleshooting" section:
     - Common DSM API errors (403, 119, 2301) with fixes
     - DSM 7.1.x member_list regression (link to issue #54)
     - FileStation upload auth quirk (SynoToken in URL, session as cookie)

6.4  Add integration test section (expanded):
     - Prerequisites: DSM account config (from AGENTS.md §Known Blocker)
     - How to run: full command with env vars
     - How to read the report: --report flag
     - How to add new integration tests
```

---

## 7. LICENSE

### Current state
- MIT, "Copyright (c) 2026 BY-SYSTEMS"
- Extensive liability disclaimer (good)
- No individual author
- No SPDX headers in source files

### Proposed changes

```
7.1  Update LICENSE copyright line:
     Copyright (c) 2026 BY-SYSTEMS SRL — Youssef Boujraf

7.2  Add SPDX headers to all .py source files:
     # SPDX-License-Identifier: MIT
     (First line of every .py file in src/synology_dsm/)
     NOTE: SPDX header is the license identifier ONLY — not the copyright line.
     Copyright stays in LICENSE file. Do not duplicate copyright in source files.

7.3  Add repo URL to LICENSE header (standard practice):
     https://github.com/by-openclaw/lib-synology-dsm
```

---

## 8. Coverage Threshold

### Current state
- pyproject.toml: `fail_under = 80`
- PR template: "80% minimum"
- README: "100% coverage"
- AGENTS.md: "283 passing, 100% coverage"

### Proposed — decision needed

```
Option A: Keep 80% as CI gate, 100% as aspiration
  - 80% prevents regressions
  - 100% is documented as current achievement, not requirement
  - If coverage drops below 100%, it's visible but not blocking

Option B: Raise CI gate to 95%
  - Allows minor drift (new untested edge case)
  - Catches real regressions
  - 100% in README stays as "current" not "required"

Option C: Raise CI gate to 100%
  - Strict — any untested line blocks merge
  - Fragile — hard to maintain as codebase grows
  - Matches README claim exactly

DECISION: 100% — this is a library others depend on.
If a line can't be tested, developer adds # pragma: no cover with a comment explaining why.
This forces conscious decisions — no silent untested code.
Update pyproject.toml fail_under = 100, PR template, README stays accurate.
```

---

## 9. Architecture Diagram Update

### Current state
- `assets/diagrams/lib-architecture.puml` — updated to v0.9.x (includes quota, storage, bandwidth, traffic control)
- `assets/exports/lib-architecture.png` — rendered
- Missing: exception hierarchy, return dict flow, credential provider flow

### Proposed changes

```
9.1  Update lib-architecture.puml to v0.10.x:
     - Add ensure() method signatures on all managers
     - Add return dict contract annotation: {"changed": bool, "action": str}
     - Add TrafficControlManager.ensure_rule() variant
     - Show dry_run flow path (separate from mutation path)

9.2  Add lib-exception-hierarchy.puml (new):
     - DSMError → DSMAuthError, DSMAPIError, DSMResourceNotFoundError,
       DSMPermissionError, DSMConnectionError, DSMValidationError
     - Shows which methods raise which exceptions

9.3  Add lib-credential-flow.puml (new):
     - VaultCredentialProvider → EnvCredentialProvider → Explicit
     - Auto-detect logic in get_credentials()
     - Shows where .env, VAULT_ADDR, explicit params enter

9.4  Update lib-context.puml:
     - Add Ansible collection (dashed — future consumer)
     - Add timeout/retry layer (when implemented)

9.5  Render all diagrams:
     - PlantUML → Kroki → PNG in assets/exports/
     - ASCII versions in assets/diagrams/
     - Link PNGs in README.md and docs/api-reference.md

9.6  Add assets/diagrams/README.md:
     - Index of all diagrams with description and last-updated date
```

---

## 10. Documentation Links

### Current state
- README links to docs/ files
- docs/ links to assets/exports/ for diagrams (partially)
- No diagram index

### Proposed changes

```
10.1  Add diagram section to README.md:
      - Architecture: ![Architecture](assets/exports/lib-architecture.png)
      - Context: ![Context](assets/exports/lib-context.png)
      - Link to assets/diagrams/README.md for full index

10.2  Add diagram links to docs/api-reference.md:
      - Exception hierarchy diagram in error handling section
      - Credential flow diagram in authentication section

10.3  Update docs/feature-coverage.md:
      - Cross-reference each manager with its ensure() status
      - Link to ADR-0002 and ADR-0007
```

---

## 11. PR Template Enhancement

### Current state
- Summary, type checkboxes, quality checklist (ruff, mypy, pytest, docstrings, changelog)
- 80% coverage reference

### Proposed changes

```
11.1  Add Closes #___ field (pre-filled):
      - Enforces issue linking
      - Auto-closes issue on merge

11.2  Add label enforcement reminder:
      - "Ensure at least one label is applied before requesting review"

11.3  Add ADR-0007 compliance check:
      - "If adding/modifying a mutating method: returns {"changed": bool, "action": str}"

11.4  Add RAID check:
      - "If new risk/issue found: RAID.md + GitHub Issue + Projects board (all three)"

11.5  Update coverage reference to match decision from topic 8
```

---

## 12. Project Board & Workflow Integration

### Current state
- `project-board-sync.yml` auto-adds issues/PRs to board
- No label automation
- No stale issue bot

### Proposed changes

```
12.1  Add label-on-PR workflow:
      - Auto-label PRs based on file paths:
        src/ → "lib", tests/ → "test", docs/ → "docs", .github/ → "ci"
      - Uses actions/labeler

12.2  Add stale issue workflow (optional — discuss):
      - Issues inactive for 30 days → labeled "stale"
      - Stale for 14 more days → closed with comment
      - Excluded: raid:*, priority:high labels
      - WARNING: this conflicts with "close manually" rule in OPERATING-STANDARD
      - Recommendation: skip this — manual close is the standard

12.3  Add PR size labeler (optional):
      - XS (<10 lines), S (<50), M (<200), L (<500), XL (>500)
      - Visual indicator for review effort
```

---

## 13. GPG Commit Signing

### Current state
- Some commits show "Verified" (Release Please bot), others don't (agent/human)
- No GPG key configured for Rune agent
- No signing policy documented

### Proposed changes

```
13.1  Human commits (@yboujraf):
      - Generate Ed25519 GPG key (or use existing)
      - git config --global commit.gpgsign true
      - Add public key to GitHub account → "Verified" badge on all commits
      - Store passphrase in Vaultwarden (until Vault Phase 2)

13.2  Agent commits (Rune VM):
      - Generate dedicated GPG key: "Rune <rune@by-systems.arpa>"
      - Configure on Rune VM: git config commit.gpgsign true
      - Add public key to GitHub as bot key
      - Unattended signing: gpg-agent with --allow-preset-passphrase (passphrase preset at boot/session start)
      - DECISION: passphrase required — no passphrase-less keys. If VM is compromised, attacker cannot sign without the preset passphrase expiring.

13.3  CI commits (Release Please):
      - Already signed by GitHub — "Verified" automatically
      - No action needed

13.4  Policy:
      - All commits to main MUST be signed (Verified badge)
      - Unsigned commits blocked via branch protection rule:
        Settings → Branches → main → "Require signed commits"
      - Add to OPERATING-STANDARD.md §4
```

---

## 14. GitHub → GitLab Migration Readiness

### Current state
- GitHub free plan: 2,000 CI minutes/month (sufficient for now)
- ADR-0005 (VCS strategy): GitLab CE is the target
- No GitLab CE deployed yet (Phase 5 — blocked on Layers 2-4)

### Proposed — keep CI portable

```
14.1  Nox as abstraction layer:
      - GitHub Actions calls nox sessions
      - GitLab CI will call the same nox sessions
      - CI config changes, test commands don't
      - Already addressed in topic 3

14.2  Keep GitHub as read-only mirror post-migration:
      - Redundancy (GitLab down → GitHub has the code)
      - Public visibility (if repos go public)
      - Issue history preserved
      - Mirror via git push --mirror or GitLab mirror feature

14.3  No action now:
      - GitLab deployment is Layer 5
      - Current GitHub free plan is sufficient
      - When GitLab is ready: rewrite .github/workflows/ to .gitlab-ci.yml
        (nox sessions stay identical)
```

---

## 15. Discord Integration

### Current state — confirmed

```
Channel: #releases (category: announcements)
Org: by-openclaw
Type: GitHub webhook → Discord (native parsing)
Events: release only
Direction: read-only (no agent interaction)
All 5 repos: configured ✅

No workflow files. No DISCORD_WEBHOOK secret. Pure GitHub webhook.
```

No changes needed. Confirmed working.

---

## Summary — Decision Points

| # | Topic | Decision | Status |
|---|---|---|---|
| 1.1 | Build validation in CI | Approve? | Recommended |
| 1.2 | Nox as CI runner | Approve? | Recommended — single source of truth for local + CI |
| 1.3 | SPDX header check | Approve? | Recommended |
| 1.5 | Pin actions with SHA | Approve? | Recommended — supply-chain security |
| 4.3 | RAID issue templates | Approve? | Recommended — atomic rule compliance |
| 4.4 | YAML issue forms | Approve? | Optional — nice but not blocking |
| 5.1 | Ruff D rules (docstrings) | Approve? | Recommended for public API only |
| 7.1 | LICENSE author line | "BY-SYSTEMS SRL — Youssef Boujraf" | Confirm name |
| 8 | Coverage threshold | **100%** | DECIDED — `# pragma: no cover` for conscious exceptions |
| 9.2-9.3 | New diagrams | Approve scope? | Recommended |
| 12.2 | Stale issue bot | **Skip** | Conflicts with manual close rule |
| 13.1-13.4 | GPG signing | Approve? | Recommended — require signed commits on main |
| 14 | GitLab migration | No action now | Nox keeps CI portable |
| 15 | Discord | Confirmed working | No changes |

---

## Execution order (after approval)

```
Phase 1 — Quick wins (no code changes):
  7.1   LICENSE: "Copyright (c) 2026 BY-SYSTEMS SRL — Youssef Boujraf"
  7.2   SPDX headers on all .py files
  11.1-11.5  PR template updates (coverage → 100%, Closes #, RAID check)
  9.6   Diagram index (assets/diagrams/README.md)

Phase 2 — CI hardening:
  1.1   Build validation job
  1.2   Nox as CI runner (replace raw pip+pytest)
  1.3   SPDX check job
  1.5   Pin action SHAs
  3.1-3.3  Nox sessions (build, spdx)
  8     Coverage threshold → 100% in pyproject.toml

Phase 3 — GPG signing:
  13.1  Human GPG key setup
  13.2  Rune agent GPG key setup
  13.4  Branch protection: require signed commits

Phase 4 — Diagrams:
  9.1-9.5  Update + create all diagrams (architecture, exception, credential, context)
  10.1-10.3  Link diagrams in README, api-reference, feature-coverage

Phase 5 — Templates & docs:
  4.1-4.3  New issue templates (chore, security, RAID)
  5.1-5.2  Ruff D rules + __all__
  6.1-6.4  CONTRIBUTING.md sections (new manager guide, ADR guide, troubleshooting, integration)
  12.1  Label automation

Each phase: agent commits with conventional format, updates openclaw-status.md,
archives before next phase. You audit via status file.
```

---

## Agent execution instructions

This proposal is executed by the team agent, not Claude Opus.

**How the agent uses this file:**
1. Read `OPERATING-STANDARD.md` (the rules)
2. Read `openclaw-status.md` (current state)
3. Read this file (the blueprint)
4. Execute phase by phase — one commit per logical change
5. Update `openclaw-status.md` after each phase
6. Archive status before moving to next phase

**What the agent must NOT do:**
- Apply all phases at once (one at a time, verify between phases)
- Skip GPG signing setup (Phase 3) — this is a security pillar
- Auto-close issues (OPERATING-STANDARD §7.2 — close manually after verification)
- Modify OPERATING-STANDARD.md (read-only for agents)

**What Claude Opus does:**
- Audits after each phase (when asked)
- Does NOT execute changes
- Does NOT modify agent memory, SOUL.md, or workspace files
- Reads and reports — separation of concerns

---

*Review each topic. Mark approve/reject/modify. Agent executes only what's approved.*
