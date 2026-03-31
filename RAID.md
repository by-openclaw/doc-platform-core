# RAID.md — doc-platform-core

> **Scope:** Platform-wide risks, issues, actions, and dependencies tracked at the doc-platform-core level.
> **Detailed per-component RAID:** see each repo's own RAID.md
> **Full platform RAID:** `docs/raid.md` (88+ items)

---

## Open Items

| ID | Type | Description | Severity | Owner | Status |
|---|---|---|---|---|---|
| R-PC-01 | RISK | RAID location conflict: ADR-0003 (centralized) vs ADR-0004 (per-repo hybrid) — resolved by ADR-0004 amendment | LOW | @yboujraf | ✅ Resolved (2026-03-30) |
| R-PC-02 | RISK | ADR-0006 layer status stale — parallel Layer 4 dev not documented as exception | LOW | @yboujraf | ✅ Resolved — exception documented in status.md |
| I-PC-01 | ISSUE | Gap report (lib-synology-dsm-gap-report-2026-03-29.md) was superseded | LOW | Rune | ✅ Resolved — archived 2026-03-30 |
| I-PC-02 | ISSUE | No live platform status view | MEDIUM | Rune | ✅ Resolved — docs/status.md created 2026-03-30 |
| R-PC-03 | RISK | ADR-0010 naming convention not yet enforced across all repos — service accounts, groups, and env labels have gaps | MEDIUM | @yboujraf | 🔄 Open — tracked per-repo in RAID.md |
| R-PC-04 | RISK | CLAUDE.md/AGENTS.md did not reference ADR-0010/ADR-0012 | LOW | Rune | 🔄 In Progress — fixing in sprint Block 3 |
| D-PC-01 | DEPENDENCY | doc-platform-core templates must stay in sync with per-repo audits | MEDIUM | @yboujraf | 🔄 Open — Tier 2 backport pending |

---

## Resolved Items

See `docs/raid.md` for the full historical RAID register (88+ items).
