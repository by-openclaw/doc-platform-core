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

ADR-0004 (per-repo documentation) was accepted. RAID tracking is now hybrid:
- Repo-scoped items → each repo's own `RAID.md`
- Cross-repo / platform items → `doc-platform-core/docs/raid.md` (this file's scope)

See: lib-synology-dsm ADR-0004 for the full decision record.
