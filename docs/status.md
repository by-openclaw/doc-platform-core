# Platform Status

> **Last updated:** 2026-03-31
> **Maintained by:** Rune — update after any release, layer change, or RAID update
> **Reading order:** status.md → roadmap.md → stack.md → relevant repo CLAUDE.md

---

## Layer Progress

| Layer | Name | Status | Completion | Blockers |
|---|---|---|---|---|
| 0 | Standards & Templates | ✅ In progress | ~90% | RAID location hybrid model (ADR-0004 adopted, ADR-0003 amended) |
| 1 | Proxmox Base | ✅ In progress | ~85% | OPNsense VM not deployed, vmbrPOC bridge missing |
| 2 | Vault | ❌ Not started | 0% | Blocked on Layer 1 completion |
| 3 | Identity (Authentik) | ❌ Not started | 0% | Blocked on Layer 2 |
| 4 | Storage (Synology) | 🔄 Active | ~95% | lib-synology-dsm: integration tests pending live NAS run, then v1.0 |
| 5 | PoC Services | ❌ Not started | 0% | Blocked on Layers 2–4 |

> **Layer 4 parallel development:** lib-synology-dsm has no infra runtime dependency on Vault/Identity.
> Development continues in parallel without violating the layer rule spirit. Documented exception.

---

## Per-Repo Status

| Repo | Version | Tests | Key Blockers |
|---|---|---|---|
| lib-synology-dsm | v0.9.3 → v1.0 pending | 345 unit, 100% coverage, ruff+mypy clean | Integration tests pending live NAS run — **awaiting team approval** |
| infra-terraform-proxmox | — | n/a | OPNsense VM scaffold, vmbrPOC bridge |
| ansible-platform | — | n/a | Hardening roles only — NetBox/Vault roles not started |
| doc-platform-core | — | n/a (docs only) | Layer 0 gap: RAID hybrid model |
| platform-setup | — | n/a | Tracks issues only |

---

## Top Blockers (from RAID)

| # | Item | Type | Priority | Repo |
|---|---|---|---|---|
| 1 | lib-synology-dsm: integration tests need live NAS run before v1.0 tag | DEPENDENCY | HIGH | lib-synology-dsm |
| 2 | lib-synology-dsm: team approval required before push + PR | ACTION | HIGH | lib-synology-dsm |
| 3 | OPNsense VM not deployed — blocks Layer 1 completion | ISSUE | HIGH | infra-terraform-proxmox |

---

## Recent Updates

| Date | What changed |
|---|---|
| 2026-03-31 | lib-synology-dsm: SharePermissionManager + SystemManager — 345 unit tests, ruff+mypy clean. Awaiting integration run + approval. Review: `docs/open-status.md` |
| 2026-03-30 | Full Tier 1 audit remediation applied — CLAUDE.md, AGENTS.md, CONTRIBUTING.md, ADRs 0004/0005/0007, SOUL.md cleaned, status.md created, gap report archived, RAID.md updated |
| 2026-03-30 | project-board-sync workflow live on all 5 repos, GH_TOKEN rotated and set org-wide |
| 2026-03-30 | lib-synology-dsm audit (16 files, 68 action items) completed by team |
| 2026-03-29 | lib-synology-dsm v0.8.0 shipped — 161 unit tests, 100% coverage, all managers have ensure() + dry_run |
| 2026-03-29 | ansible-platform repo created — hardening role (sshd, fail2ban, ufw, postfix) |
| 2026-03-28 | infra-terraform-proxmox — first VM deployed, baseline validated (vm-debian-bootstrap-test-01) |
| 2026-03-28 | ADR-0006 Platform Charter accepted |
| 2026-03-27 | lib-synology-dsm v0.6.1 — FileStation complete |

---

## How to Update

After any release or layer change:
1. Update the relevant row in "Per-Repo Status"
2. Update "Layer Progress" if completion % changed
3. Update "Top Blockers" — add/remove as RAID changes
4. Add a row to "Recent Updates"
5. Commit: `docs: update platform status to YYYY-MM-DD`
