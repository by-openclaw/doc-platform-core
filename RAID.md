# RAID.md — `doc-platform-core`

> **Scope:** Platform-wide risks, issues, actions, and dependencies tracked at the `doc-platform-core` level.
> **Detailed per-component RAID:** see each repo's own `RAID.md`
> **Full platform RAID:** [`docs/raid.md`](docs/raid.md)

---

## Open items

| ID | Type | Description | Severity | Owner | Status |
|---|---|---|---|---|---|
| D-PC-01 | DEPENDENCY | `doc-platform-core` templates (`docs/templates/*.tpl.md`) must stay in sync with per-repo `AGENTS.md`, `CLAUDE.md`, `CONTRIBUTING.md` improvements — backport pattern | MEDIUM | @yboujraf | 🔄 Open |

---

## Resolved items

| ID | Type | Description | Resolved |
|---|---|---|---|
| R-PC-01 | RISK | RAID location conflict: centralized vs per-repo hybrid — resolved via per-repo `RAID.md` + platform-wide `docs/raid.md` split, now captured in `OPERATING-STANDARD.md §3.2 File ownership` and `§7 Issue & Project Tracking` | 2026-03-30 |
| R-PC-02 | RISK | Layer status stale — parallel Layer 4 library dev not documented as exception | 2026-03-30 — exception documented in `docs/adr/infra/0002-platform-charter §Parallel development exception` |
| I-PC-01 | ISSUE | Gap report (`lib-synology-dsm-gap-report-2026-03-29.md`) was superseded | 2026-03-30 — archived |
| I-PC-02 | ISSUE | No live platform status view | 2026-03-30 — `docs/status.md` created |
| R-PC-03 | RISK | Naming convention not yet enforced across all repos — service accounts, groups, env labels had gaps | 2026-04-14 — naming is now enforced by 4 scoped ADRs: `naming/0001-infra`, `naming/0002-identity`, `naming/0003-firewall`, `naming/0004-automation`. Stability rule in `naming/0001-infra §10` forbids env in hostnames. Per-repo `CLAUDE.md` updates are tracked as a separate follow-up. |
| R-PC-04 | RISK | `CLAUDE.md` / `AGENTS.md` did not reference ADR-0010 / ADR-0012 | 2026-04-14 — top-level metadata files refreshed in PR #39 to reference scoped ADRs (`naming/0001-infra`, `infra/0005-environment-tiers`). Per-repo CLAUDE.md cross-repo updates remain as a follow-up. |

---

## Full history

See [`docs/raid.md`](docs/raid.md) for the full platform-wide RAID register, which tracks items that span multiple repos or the platform as a whole. This file holds only items specific to `doc-platform-core` itself.
