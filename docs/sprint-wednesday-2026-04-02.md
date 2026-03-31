# Sprint: Finalize Infrastructure — Deadline Wednesday 2026-04-02

> **Owner:** @yboujraf
> **Executor:** Team agent (Rune)
> **Auditor:** Claude Opus (on request)
> **Status:** PLANNING — approve before agent starts

---

## Goal

By Wednesday EOD: all infra conventions locked, all agent files aligned, all ADRs written, compliance tags on everything. No more back-and-forth on naming, identity, secrets, or structure.

---

## What exists today (done)

| Item | Status |
|---|---|
| OPERATING-STANDARD.md | ✅ Written, §6.1 updated with secrets chain |
| 18 audit files + 4 per-repo audits | ✅ Written |
| O1–O10 cross-repo optimization | ✅ Executed |
| Secrets JSON schema + ADR-0009 | ✅ Written |
| Password redaction (~30 files) | ✅ Done |
| Naming & identity convention draft | ✅ Written (doc-platform-core/docs/) |
| CI/platform proposal (15 topics) | ✅ Written (doc-platform-core/docs/) |
| v2 PoC design doc (BaseManager ABC) | ✅ Written (lib-synology-dsm/docs/) |
| lib-synology-dsm at 345 tests, v1.0 candidate | ✅ Agent work |

## What needs to happen by Wednesday

### Block 1 — ADRs (lock decisions, stop re-discussing)

| # | ADR | Repo | Content | Compliance tags |
|---|---|---|---|---|
| B1.1 | ADR-0008: Method naming convention | lib-synology-dsm | verb() for primary, verb_entity() for sub-entity, load/save for UoW | — |
| B1.2 | ADR-00XX: Naming & identity convention | doc-platform-core | Env labels, account types, groups, lifecycle — from draft | ISO A.9.1.1, A.9.2.1, NIS2 Art.21 |
| B1.3 | ADR-00XX: Compliance framework mapping | doc-platform-core | NIS2 + ISO 27001 + GDPR → BY-SYSTEMS controls | All |
| B1.4 | ADR-00XX: Secret storage convention | doc-platform-core | Vault KV v2 JSON locally, one-file-per-env, migration path | ISO A.10.1.1, NIS2 Art.21(2)(d) |
| B1.5 | ADR-00XX: Environment tier standard | doc-platform-core | poc/dev/test/staging/acc/prod — explicit always, no implicit prod | ISO A.12.1.4 |

### Block 2 — Agent files (align all repos)

| # | Task | Repos | What |
|---|---|---|---|
| B2.1 | Update OPERATING-STANDARD.md | workspace | Add §6.5 naming/identity reference, add compliance control tags to all sections |
| B2.2 | Update CLAUDE.md in all repos | all 5 | Reference naming convention ADR, add env constraint |
| B2.3 | Update AGENTS.md stats | all 5 | Current version, test count, file count |
| B2.4 | Update openclaw-status.md | workspace | Decisions Made section — add all new ADRs |
| B2.5 | Update RAID.md | all repos with one | Flag any gap between convention and current state |

### Block 3 — Compliance matrix (first pass)

| # | Task | Repo | What |
|---|---|---|---|
| B3.1 | Create compliance-matrix-draft.md | doc-platform-core | Map ISO 27001 Annex A + NIS2 + GDPR to BY-SYSTEMS implementation |
| B3.2 | Tag existing ADRs with compliance controls | doc-platform-core + lib-synology-dsm | Add `## Compliance` section to each ADR |
| B3.3 | Tag OPERATING-STANDARD.md sections | workspace | Each section gets ISO/NIS2/GDPR control reference |
| B3.4 | Add CISO Assistant to stack.md | doc-platform-core | Tier 1 tool — GRC dashboard |

### Block 4 — Templates (stop reinventing)

| # | Task | Repo | What |
|---|---|---|---|
| B4.1 | Create ADR template with compliance section | doc-platform-core/docs/templates/ | ADR-0000 + `## Compliance` field |
| B4.2 | Create secret JSON template | doc-platform-core/docs/templates/ | Vault KV v2 schema with all fields |
| B4.3 | Create RAID.md template | doc-platform-core/docs/templates/ | Risks/Issues/Dependencies tables |
| B4.4 | Create per-repo review checklist | doc-platform-core/docs/templates/ | Agent reads this before declaring "done" |

---

## Review checklist template (B4.4) — the "no more 1000x review" solution

Before any agent declares a repo "done," run this checklist:

```markdown
## Repo Review Checklist — {repo-name}

### Files exist
- [ ] README.md — current, reflects actual state
- [ ] CLAUDE.md — hard rules, v1.0 blockers, current state updated
- [ ] AGENTS.md — stats current, reading order correct
- [ ] CONTRIBUTING.md — setup, commit, DoD, diagrams, post-release
- [ ] CHANGELOG.md — up to date
- [ ] SECURITY.md — supported version current
- [ ] RAID.md — repo-scoped risks, no stale items
- [ ] LICENSE — correct entity + author
- [ ] docs/adr/ — all decisions recorded, index updated
- [ ] docs/archive/ — directory exists
- [ ] .github/ISSUE_TEMPLATE/ — bug + feature (+ RAID if applicable)
- [ ] .github/PULL_REQUEST_TEMPLATE.md — checklist matches DoD
- [ ] .github/CODEOWNERS — owner set
- [ ] .github/workflows/release-please.yml — working
- [ ] .github/workflows/project-board-sync.yml — working

### Content aligned
- [ ] CLAUDE.md references OPERATING-STANDARD.md
- [ ] CLAUDE.md references naming convention ADR
- [ ] AGENTS.md and CLAUDE.md are not duplicated (dedup pattern applied)
- [ ] Secret references use <REDACTED:{type}> — no plaintext credentials
- [ ] Environment labels follow convention (poc/dev/test/staging/acc/prod)
- [ ] Service accounts follow naming (svc-{function}-{env})
- [ ] All ADRs have ## Compliance section with ISO/NIS2/GDPR tags

### Quality (repo-type dependent)

**Type: `lib`** (lib-synology-dsm)
- [ ] CI green
- [ ] Tests pass
- [ ] Ruff clean
- [ ] Mypy clean
- [ ] Coverage at 100% (`# pragma: no cover` for conscious exceptions)

**Type: `infra`** (infra-terraform-proxmox, ansible-platform, platform-setup)
- [ ] CI green (if CI exists)
- [ ] `terraform fmt` / `ansible-lint` clean (if applicable)
- [ ] No coverage or linting requirement — these are IaC/config repos

**Type: `docs`** (doc-platform-core)
- [ ] Structure only — no CI quality gates
- [ ] Markdown renders correctly (no broken links)
- [ ] File naming follows `docs/naming-convention.md`

### Compliance
- [ ] No plaintext secrets in any committed file
- [ ] RAID.md reflects current risks
- [ ] ADRs tagged with compliance controls
- [ ] Account naming follows convention
```

---

## Execution order

```
Day 1 (Tuesday 2026-03-31 — today):
  You: finalize naming convention decisions (svc-rune vs rune-api, adm_ underscore)
  You: approve sprint plan after team feedback applied
  Compliance tags on naming draft: done (2026-03-31, applied by Claude Opus)

Day 2 (Tuesday 2026-04-01):
  Agent: Block 1 (ADRs) + Block 4 (templates) — in parallel
  Agent: Block 2 (agent files) — after B1 lands (B2 references new ADRs)
  You: review at EOD

Day 3 (Wednesday 2026-04-02):
  Agent: Block 3 (compliance matrix) — first pass
  Agent: run review checklist on all 5 repos
  You/Claude Opus: final audit
  You: approve → lock → done
```

---

## After Wednesday

- No more re-discussing naming, identity, secrets, or env conventions — they're in ADRs
- New repos: copy templates, fill in placeholders, run checklist
- New ADRs: use template with compliance section
- Agent reviews: run checklist, report pass/fail
- CISO Assistant: imported compliance matrix when deployed

---

*Approve this plan → agent starts Block 1 tomorrow.*
