# CLAUDE.md — {{REPO_NAME}}

> **Scope:** `{{SCOPE}}` | **Component:** `{{COMPONENT}}`
> **GitHub:** `by-openclaw/{{REPO_NAME}}`
> **Layer:** Layer {{LAYER_NUMBER}} — {{LAYER_NAME}} (ADR-0006)

AI agent context. Read before touching any file.

---

## What This Repo Does

<!-- One paragraph max. What is this repo for? NOT for? -->
{{DESCRIPTION}}

---

## Hard Rules

<!-- Non-negotiable architectural decisions. Reference ADRs. -->
<!-- Example: -->
<!-- ### HTTP client — urllib only (ADR-0001) -->
<!-- - NEVER use httpx, requests, or any third-party HTTP library. -->

> Move these to the top. An agent that reads 10 lines and stops should know the guardrails.

---

## v1.0 Blockers

<!-- What must be true before this repo is production-ready? -->

| Priority | Blocker | Status |
|---|---|---|
| HIGH | {{BLOCKER_1}} | OPEN |

See: RAID.md for repo-scoped risks.

---

## Current State (v{{VERSION}} — {{DATE}})

<!-- Volatile — update after every release. -->

| Component | Status |
|---|---|
| {{COMPONENT_1}} | ✅ / ❌ / ⏸ |

---

## Open Issues

| Priority | Issue | Tracking |
|---|---|---|
| {{PRIORITY}} | {{ISSUE}} | {{STATUS}} |

---

## Key Files

| File | Why |
|---|---|
| `README.md` | Install, quickstart |
| `CONTRIBUTING.md` | Dev setup, commit format, DoD checklist |
| `AGENTS.md` | Agent onboarding, coding standards, project stats |
| `RAID.md` | Repo-scoped risks, issues, dependencies |

---

## Constraints

<!-- Non-negotiable rules specific to this repo. -->
<!-- For coding standards, commit format, and operational guardrails → see AGENTS.md -->

---

## Diagram Standard

See ADR-0006 §9. Source → `assets/diagrams/`, render → `assets/exports/`, commit + post to Discord.

---

## Related

- Platform charter: `doc-platform-core/docs/adr/0006-platform-charter.md`
- RAID (platform-wide): `doc-platform-core/docs/raid.md`
- Refactor decisions: `docs/refactor-clarification-*.md` (if applicable)
