# ADR-0006 — Platform Charter

**Date:** 2026-03-28
**Status:** Accepted — gates all future work
**Scope:** All BY-SYSTEMS PoC platform work, all repos, all agents
**Authors:** yboujraf + Rune
**Supersedes:** Scattered decisions from 2026-03-25 → 2026-03-28

> **Rule #1:** Nothing proceeds to the next layer until the current layer is documented, tested, and signed off.
> **Rule #2:** Every decision lives in an ADR. Every finding lives in RAID + GitHub Issue + Project board.
> **Rule #3:** AI agents assist. They do not decide. They do not skip layers.

---

## 1. Work Layers — Strict Sequential Order

```
┌─────────────────────────────────────────────────────────────────┐
│  Layer 0 — Standards & Templates          ← THIS ADR            │
│  Define once. Apply forever. No exceptions.                      │
├─────────────────────────────────────────────────────────────────┤
│  Layer 1 — Proxmox Base                                         │
│  Cloud-init template finalized. VM/LXC baseline locked.         │
│  Acceptance: any VM provisioned = same baseline, zero manual     │
├─────────────────────────────────────────────────────────────────┤
│  Layer 2 — Vault                                                │
│  Secrets management. Nothing deploys credentials without Vault. │
│  Acceptance: Vault cluster up, PKI + KV mounted, policies set   │
├─────────────────────────────────────────────────────────────────┤
│  Layer 3 — Identity (DevOps users + Authentik)                  │
│  Authentik OIDC, devops service accounts, SSH key distribution. │
│  Acceptance: all users from Vault, SSH via CA, no static keys   │
├─────────────────────────────────────────────────────────────────┤
│  Layer 4 — Storage (Synology)                                   │
│  NFS shares, CRUD API, permissions via lib-synology-dsm.        │
│  Acceptance: lib tested, CI green, shares provisioned via code  │
├─────────────────────────────────────────────────────────────────┤
│  Layer 5 — PoC Services                                         │
│  NetBox, GitLab CE, monitoring, etc.                            │
│  Acceptance: each service has runbook, CI, RAID entry           │
└─────────────────────────────────────────────────────────────────┘
```

**Current status (2026-03-28):** Layer 0 in progress. Layer 1 at 80%.
**Next action:** Complete Layer 0 (this charter + templates), finalize Layer 1.

---

## 2. Network & IP Addressing Plan

### PoC Segment — Isolated from Production

```
Production fabric (10.6.0.0/20):   UNTOUCHED — no changes
PoC WAN path: ISP → pfSense OOB → vmbrOOB → OPNsense VM → vmbrPOC
```

### IP Address Plan

| VLAN | Purpose | Subnet | Mask | Usable Range | GW |
|---|---|---|---|---|---|
| OOB (prod) | Infrastructure mgmt | 10.6.224.0 | /24 | .1 – .254 | 10.6.224.1 |
| POC-MGMT | PoC VM management | 10.6.225.0 | /24 | .1 – .254 | 10.6.225.1 |
| POC-SVC | PoC services | 10.6.226.0 | /24 | .1 – .254 | 10.6.226.1 |
| POC-DHCP | Dynamic (dev/test) | 10.6.239.0 | /24 | .101 – .199 | 10.6.239.1 |
| MGMT | Platform orchestration | 10.6.240.0 | /20 | .1 – .254 | 10.6.224.1 |

**Static assignment table (PoC):**

| Host | IP | VLAN | Purpose |
|---|---|---|---|
| srv-proxmox-poc-01 | 10.6.224.105 | OOB | Hypervisor |
| vm-opnsense-poc-01 | 10.6.225.1 | POC-MGMT | PoC firewall/GW |
| vm-netbox-poc-01 | 10.6.226.10 | POC-SVC | CMDB |
| vm-vault-poc-01 | 10.6.226.11 | POC-SVC | Secrets (Layer 2) |
| vm-authentik-poc-01 | 10.6.226.12 | POC-SVC | Identity (Layer 3) |
| vm-gitlab-poc-01 | 10.6.226.20 | POC-SVC | VCS/CI (future) |
| rune-vm | 10.6.225.10 | POC-MGMT | Automation/OpenClaw |

---

## 3. VM User Standard

Every VM/LXC has exactly two user accounts:

| User | Purpose | Sudo | Auth | Notes |
|---|---|---|---|---|
| `by-systems` | Human OOB access | yes | SSH pubkey (personal Win11 key) | Break-glass, human only |
| `rune` | Automation/agent OOB | yes | SSH pubkey (rune automation key) | Terraform, OpenClaw, CI |

**No other users at provisioning time.** Service accounts are added post-Vault via Authentik (Layer 3).

**SSH Keys:**

```
by-systems key: ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIAoUn/DYxSFLu+TDqKlQwsllWfr5G0NEVI3Jh0sn0yvm yboujraf@personal-2026-03-27
rune key:       ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIHbkOZYUkqJ9pdmDWDm87MBI1Rf4x7fZV3IMuitG+qlu rune@by-systems-rune-vm
```

**Rune VM rename task:** Current `by-systems` user on Rune VM → remains. Add `rune` user as the automation identity going forward. Document in RAID as [A] action item.

---

## 4. Repository Standard

### Naming (confirmed, from ADR-0002 + ADR-0005)

```
{category}-{scope}-{qualifier}

Categories:   doc  platform  infra  lib  svc  k8s  ci
Scope:        kebab-case, English, no abbreviations except established ones
```

### Every repo MUST contain on creation:

```
repo-root/
├── README.md              ← generated from template
├── CLAUDE.md              ← AI agent context (see §6)
├── AGENTS.md              ← sub-agent rules + spawn policy
├── CHANGELOG.md           ← managed by release-please / commitizen
├── .commitlintrc.json     ← commitizen config (conventional commits)
├── .releaserc.json        ← semantic-release config
├── .gitattributes         ← LFS rules for binary files
├── assets/
│   ├── diagrams/          ← .puml source files
│   └── exports/           ← rendered PNG/SVG (linked from docs)
├── docs/
│   ├── adr/               ← Architecture Decision Records
│   └── runbooks/          ← Operational procedures
└── .github/
    └── ISSUE_TEMPLATE/    ← RAID-tagged issue templates
```

### Commit Convention (Conventional Commits)

```
<type>(<scope>): <subject>

Types: feat | fix | docs | chore | refactor | test | ci | perf | security
Scope: component name, optional

Examples:
  feat(vm-linux): add cloud-init locale support
  fix(proxmox): correct SCSI controller for iothread
  docs(adr): add ADR-0006 platform charter
  security(vault): rotate terraform token
```

### Semantic Versioning

```
MAJOR.MINOR.PATCH

MAJOR: breaking change (infra: new network segment, app: API break)
MINOR: new feature (infra: new VM type, app: new endpoint)
PATCH: fix/docs/chore

Tags: v1.2.3
Infra repos use PATCH for config changes, MINOR for new module/component
```

### Branch Strategy

```
main         ← always deployable, protected, PR required
dev          ← integration branch
feature/*    ← feature branches, merge → dev
hotfix/*     ← urgent fixes, merge → main + dev
release/*    ← release prep
```

---

## 5. RAID Tracking — Atomic Rule

Every issue, risk, dependency, or gap detected during any work triggers **three simultaneous actions:**

```
1. doc-platform-core/docs/raid.md entry
2. GitHub Issue on by-openclaw/platform-setup
   - Labels: raid:risk | raid:action | raid:issue | raid:dependency
   - Label: phase:N (1/2/3) + priority:high|medium|low
3. GitHub Projects board (#1) — add to correct column
```

**No partial tracking. All three or none.**

### GitHub Issue Labels (RAID)

| Label | Meaning |
|---|---|
| `raid:risk` | R — Risk |
| `raid:action` | A — Action required |
| `raid:issue` | I — Issue |
| `raid:dependency` | D — Dependency / blocker |

---

## 6. CLAUDE.md Standard (per repo)

Every repo's `CLAUDE.md` must contain:

```markdown
# CLAUDE.md — {repo-name}

## What this repo does
One paragraph. What it is, what it is NOT.

## Layer
Layer N — {name}  (from ADR-0006 §1)

## Key files (always read first)
Table: file | why it matters

## Versions
Table: tool | version | notes

## Current state
Table: component | status (✅ done / ⏸ planned / 🔴 blocked)

## Constraints (non-negotiable)
Bullet list of things the agent must never do without explicit approval.

## Diagram standard
(copy from ADR-0006 §8 — mandatory)

## RAID
Link to raid.md + open issues relevant to this repo.

## Related repos / links
```

---

## 7. Sub-Agent Rules — Non-Blocking Pattern

### Problem
Long-running sub-agents (Terraform, VM provisioning, lib builds) block the Discord ↔ OpenClaw connection and require gateway restarts.

### Solution: Background + Cron pattern

```
Long task (>30s estimated):
  1. Rune spawns sub-agent with background=true
  2. Sub-agent writes progress to workspace file: workspace/state/{task-id}.json
  3. Cron job polls state file every 60s
  4. When done: cron fires system event → Rune reports to Discord
  5. Gateway never blocks

Short task (<30s): inline, normal flow
```

### State file format

```json
{
  "task_id": "terraform-vm-netbox-poc-01",
  "status": "running|done|failed",
  "started": "2026-03-28T19:00:00Z",
  "updated": "2026-03-28T19:05:00Z",
  "summary": "Applied 3 of 5 resources...",
  "error": null
}
```

### Timeout rules

| Operation | Timeout | On timeout |
|---|---|---|
| Terraform apply | 10 min | write state=failed, report to Discord |
| VM SSH ready check | 5 min | retry 3x then fail |
| API calls (Synology, Proxmox) | 30s | retry 2x then fail |
| Sub-agent (general) | 15 min | kill + report partial |

---

## 8. AI Model Routing

| Task type | Primary | Fallback | Reasoning |
|---|---|---|---|
| Code generation, refactor | `anthropic/claude-sonnet-4-6` | `openai-codex/gpt-5.4` | Claude stronger on complex reasoning |
| Terraform, IaC | `anthropic/claude-sonnet-4-6` | `openai-codex/gpt-5.4` | Claude better at structured outputs |
| Diagram generation (text) | `anthropic/claude-sonnet-4-6` | — | PlantUML via Kroki, no image model needed |
| Image generation (marketing) | `google/imagen-4-fast` | — | Requires Gemini API key (see runbook) |
| Security audit, threat model | `anthropic/claude-sonnet-4-6` | — | Best structured reasoning |

**Current gap:** No image model configured. Gemini API key needed for marketing diagrams.
**Runbook:** `platform-setup/tools/openclaw/runbooks/gemini-image-gen-setup.md`

---

## 9. Diagram Standard (mandatory for all agents)

### Technical diagrams (default, always)

```
1. Write source → assets/diagrams/{type}-{subject}-v{N}.puml
2. Render PNG → assets/exports/{type}-{subject}-v{N}.png  (via Kroki)
3. Write ASCII → assets/diagrams/{type}-{subject}-v{N}-ascii.txt
4. Commit all three to repo
5. Post PNG to Discord #bot-openclaw (channel 1486559640945819828)
6. Docs link to assets/exports/ ONLY
```

### Marketing diagrams (when explicitly requested)

```
1. Generate via google/imagen-4-fast (requires API key)
2. Save to assets/exports/{type}-{subject}-marketing-v{N}.png
3. Commit via LFS
4. Post to Discord
```

### Naming

```
{type}-{subject}-v{N}.{ext}

Types: arch | seq | flow | network | erd | component | deployment
Examples:
  arch-poc-infra-v1.puml
  flow-vm-provisioning-v1.puml
  network-poc-vlan-v1.puml
  seq-vault-auth-v1.puml
```

---

## 10. Open Actions (RAID)

| # | Type | Description | Owner | Priority |
|---|---|---|---|---|
| A-001 | A | Add `rune` OS user to Rune VM, update Terraform cloud-init | Rune | high |
| A-002 | A | Finalize cloud-init template (locale be, tz Brussels, both SSH keys, `rune` user) | Rune | high |
| A-003 | A | Create GitHub Issue templates for RAID labels | Rune | medium |
| A-004 | A | Apply this charter to all existing repos (CLAUDE.md, AGENTS.md, assets/ structure) | Rune | high |
| A-005 | A | Deploy vm-netbox-poc-01 (blocked pending Layer 1 finalization) | Rune | medium |
| A-006 | A | Deploy vm-vault-poc-01 (Layer 2) | Rune | medium |
| R-001 | R | No Vault yet — all secrets stored as files/env vars (acceptable for PoC phase only) | — | high |
| D-001 | D | Gemini API key needed for marketing image gen | yboujraf | low |
| D-002 | D | Synology rune-api user needs Application→DSM=Allow | yboujraf | medium |

---

## 11. Contribution Flow (for other contributors)

```
                    ┌─────────────────────────────────┐
                    │      CONTRIBUTION FLOW           │
                    └─────────────────────────────────┘

  1. Read this ADR (ADR-0006)
  2. Read the repo's CLAUDE.md and AGENTS.md
  3. Check open RAID items in platform-setup GitHub Issues
        ↓
  4. Create feature branch:  feature/{issue-id}-{short-desc}
  5. Commit with conventional commits
        ↓
  6. If new finding → RAID.md + GitHub Issue + Projects board (all three)
  7. If architecture decision → new ADR in doc-platform-core/docs/adr/
        ↓
  8. PR → dev branch → review → merge
  9. Release → semantic version tag → CHANGELOG updated
        ↓
  10. Post diagram/summary to Discord #bot-openclaw

  AI agents follow the same flow.
  AI agents do not merge PRs without human approval.
  AI agents do not proceed past current layer without sign-off.
```

---

## Acceptance Criteria (this ADR is "done" when)

- [ ] All existing repos have `CLAUDE.md` updated to this standard
- [ ] All existing repos have `AGENTS.md`
- [ ] All existing repos have `assets/` structure + `.gitattributes`
- [ ] GitHub Issue templates created with RAID labels
- [ ] A-001: `rune` user added to Rune VM
- [ ] A-002: cloud-init template finalized and re-validated

---

## References

- ADR-0001: Platform Stack
- ADR-0002: Repository Structure
- ADR-0003: RAID Issue Tracking
- ADR-0004: Identity & SSO
- ADR-0005: VCS & CI/CD Strategy
- Naming convention: `docs/naming-convention.md`
- RAID tracker: `docs/raid.md`
- Gemini runbook: `platform-setup/tools/openclaw/runbooks/gemini-image-gen-setup.md`
