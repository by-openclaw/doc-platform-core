# Audit: doc-platform-core — Agent Files & Repo Health

> **Audited:** 2026-03-30
> **Auditor:** Rune (via Claude Opus)
> **Version:** v0.5.0 | **Commits:** 54 | **Files:** 38

---

## What exists

| File | Exists | Status |
|---|---|---|
| CLAUDE.md | Yes | Good — clear "docs only" scope, commit type restrictions |
| AGENTS.md | Yes | Good — reading order, ADR format, docs-only constraints |
| README.md | Yes | Good — full contents listing with links |
| CONTRIBUTING.md | **No** | Missing (template exists at docs/templates/CONTRIBUTING.tpl.md) |
| CHANGELOG.md | Yes | v0.5.0, well-maintained |
| SECURITY.md | **No** | Missing — docs repo, low priority |
| RAID.md (root) | **No** | Missing at root (exists at docs/raid.md) |
| docs/adr/ | Yes | 8 ADRs (0001–0008) — strong |
| docs/audits/ | **No** | Created now |
| docs/raid.md | Yes | 88+ items — comprehensive |
| docs/status.md | Yes | Recently created |
| docs/templates/ | Yes | 4 templates (CLAUDE, CONTRIBUTING, README, SOW) |
| docs/archive/ | Yes | 2 files (brainstorm, gap report) |
| .github/CODEOWNERS | Yes | Exists |

---

## What's good

- **Strongest ADR collection** — 8 platform-wide ADRs, all accepted
- CLAUDE.md correctly restricts to docs-only commit types (docs, chore, fix)
- Templates exist for standardizing other repos
- RAID.md is comprehensive with 88+ tracked items
- docs/status.md recently created (fills the "where are we now" gap from audit 15)
- Archive directory exists and is being used

---

## What's missing

| # | Item | Priority | Why |
|---|---|---|---|
| M1 | CONTRIBUTING.md | MEDIUM | Template exists but not instantiated. Docs repo — lower priority than code repos. |
| M2 | Global ADR index (README.md in docs/adr/) | HIGH | 8 ADRs exist but no index/README in the adr/ directory |
| M3 | docs/status.md verification | MEDIUM | Recently created — verify it reflects current layer progress |
| M4 | Gap report archived? | MEDIUM | lib-synology-dsm-gap-report-2026-03-29.md — already in archive/ |
| M5 | `add-to-project` workflow | MEDIUM | Board automation — verify if project-board-sync.yml already added |
| M6 | docs/adr/0008 password | HIGH | Line 54 contains `BySyst3ms_` in API login example → **REDACT** |

---

## CLAUDE.md / AGENTS.md issues

| Item | Problem | Fix |
|---|---|---|
| CLAUDE.md scope | Good — correctly says "no code, no builds, no tests" | No action |
| AGENTS.md stats | "54 commits, 38 files" — verify | Update if stale |
| Templates vs reality | Templates exist for CLAUDE, CONTRIBUTING, README — but actual repo files have drifted from templates | Update templates to reflect audit recommendations |
| RAID.md location | At docs/raid.md, not root — inconsistent with other repos | OK for this repo (docs/ is the content area). But if per-repo RAID is adopted, convention should be documented. |

---

## Secrets check

- **docs/adr/0008-terraform-state-management.md:54** — password `BySyst3ms_` → **REDACT**
- **docs/adr/0006-platform-charter.md:96-97** — SSH public keys → **REDACT with `<REDACTED:ssh-pubkey>`**
- docs/stack.md — **CRITICAL: Discord webhook token fully exposed** (flagged in audit 16)
- IP addresses throughout — acceptable (private repo, needed for architecture docs)

---

*Action: Redact secrets (webhook token CRITICAL, password HIGH, SSH keys MEDIUM). Create ADR index. Instantiate CONTRIBUTING.md from template. Update templates to match audit recommendations.*
