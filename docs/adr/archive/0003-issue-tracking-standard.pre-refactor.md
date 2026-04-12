# ADR-0003: Issue Tracking Standard — RAID + GitHub Projects

**Date:** 2026-03-26
**Status:** Accepted
**Deciders:** yboujraf, Rune

## Context
Infrastructure audits and ongoing work generate findings that need to be tracked across sessions. Without a systematic approach, issues get lost between agent sessions.

## Decision
Any detected issue, risk, gap, or dependency must be tracked in two places:

1. **doc-platform-core/docs/raid.md** — durable record with context, severity, and mitigation
2. **by-openclaw/platform-setup GitHub Issues** — actionable ticket with steps and reference to audit file
3. **GitHub Projects board** (https://github.com/orgs/by-openclaw/projects/1) — for phase/priority tracking

This applies to:
- Infrastructure audit findings
- Security gaps
- Configuration inconsistencies
- Missing components
- Anything flagged ⚠️ or 🔴 in any audit or runbook

## Consequences
- Nothing gets lost between sessions
- All findings have a paper trail
- RAID.md serves as the compliance evidence log (NIS2/ISO 27001)
- GitHub Issues serve as the work queue

## Amendment (2026-03-30)

Per-repo documentation standard was accepted. RAID tracking is now hybrid:
- Repo-scoped items → each repo's own `RAID.md`
- Cross-repo / platform items → `doc-platform-core/docs/raid.md` (this ADR's scope)

Each repo's own per-repo documentation ADR defines its RAID scope independently.

## CISO mapping

> Applies only to controls directly relevant to this ADR's scope.

### ISO/IEC 27001:2022

| Control | Title | Status | Notes |
|---|---|---|---|
| A.5.27 | Learning from information security incidents | ✓ Covered | All findings tracked in RAID.md with severity and mitigation |
| A.8.16 | Monitoring activities | ✓ Covered | GitHub Issues + Project board provide a live work queue visible to all contributors |

### NIS2 (Directive 2022/2555)

| Article | Requirement | Status | Notes |
|---|---|---|---|
| Art. 21(2)(b) | Incident handling | ✓ Covered | RAID.md is the compliance evidence log for all detected issues and gaps |

### GDPR (Regulation 2016/679)

Not applicable — this ADR covers issue tracking process, not personal data.
