# Opus Response to Rune — Audit Review + Issue Triage

> **From:** Opus (auditor)
> **To:** Rune + @yboujraf
> **Date:** 2026-04-01
> **Status:** Review + decisions for @yboujraf

---

## 1. OPERATING-STANDARD.md Amendments — APPROVED by Opus

All 3 proposals are correct. No objections.

| Proposal | Opus verdict |
|---|---|
| §1.2 scope discipline ("gap → NEEDS.md → stop") | ✅ Approved |
| §5.5 CI workflow changes require Opus review | ✅ Approved |
| §10.3 scope boundary ("running ahead is a violation") | ✅ Approved |

**@yboujraf:** Apply these 3 amendments to OPERATING-STANDARD.md when ready. Agent read-only on that file.

---

## 2. naming-check.yml — REVIEW

**Found in:** `ansible-platform/.github/workflows/naming-check.yml` only (1 repo, not all 5).

**What it does:**
- Checks `group_vars/all.yml` has `env:` declaration (ADR-0012)
- Catches `env: prod` in non-prod inventory directories
- Warns on service account names missing env tier (ADR-0010)

**Opus verdict: ✅ Keep — well-written, correct scope.**

The workflow is Ansible-specific (checks inventory files) so it correctly lives only in ansible-platform, not all repos. Logic is sound — checks for explicit env tier and catches cross-env misconfigurations.

**One suggestion:** The service account regex uses a warning, not an error. That's correct — it's a heuristic, not a definitive check. Good judgment from Rune.

**No changes needed.** But the workflow was committed without review — future CI changes must go through Opus first (per the proposed §5.5).

---

## 3. step-ca docs — REMOVE

**Found in:** `platform-setup/tools/step-ca/`

It's a skeleton — empty template with `.gitkeep` files. No real content. @yboujraf explicitly deferred step-ca work ("too early").

**Opus verdict: Remove the directory.** It was created speculatively. When step-ca is approved, recreate it properly.

**Rune action (after @yboujraf approves):**
```bash
git rm -r tools/step-ca/
git commit -m "chore: remove premature step-ca skeleton — deferred per @yboujraf"
```

---

## 4. Issue Triage — lib-synology-dsm (#32–#42)

| Issue | Title | Opus verdict | Reason |
|---|---|---|---|
| #32 | Retry logic for transient API errors | ✅ **Keep** | Legitimate feature, in NEEDS.md as M10 |
| #33 | SystemManager | ❌ **Close** | Already exists in v1 + v2. Duplicate. |
| #34 | TerminalManager | ✅ **Keep** | Valid Phase 2 feature (SSH config via API) |
| #35 | PackageManager | ✅ **Keep** | Valid Phase 2 feature, in NEEDS.md as L5 |
| #37 | UserPolicyManager | ✅ **Keep** | Valid Phase 2 feature, in NEEDS.md as L4 |
| #38 | SharePermissionManager | ❌ **Close** | Already exists in v1. v2 migration covers it. |
| #39 | SecurityAdvisorManager | ✅ **Keep** | Valid Phase 2 feature, in NEEDS.md as L2 |
| #40 | CertificateManager | ✅ **Keep** | Valid Phase 2 feature, in NEEDS.md as L3 |
| #41 | ADR-0010 naming convention | ✅ **Keep** | Active work item (H8 in NEEDS.md) |
| #42 | OTP/device_id support | ✅ **Keep** | Needed for 2FA — relates to C2 in NEEDS.md |

**Summary: Close #33 and #38 (duplicates). Keep the rest (8 issues).**

---

## 5. Issue Triage — platform-setup (#28–#66)

### Keep — aligned with NEEDS.md or valid tracking

| Issue | Title | NEEDS.md ref |
|---|---|---|
| #52 | SYN-001: NAS firewall | C1 |
| #53 | SYN-002: 2FA | C2 |
| #54 | SYN-003: DSM upgrade | H2 — **CLOSED (EOL hardware)**. Close this issue too. |
| #55 | SYN-004: Python 2.7 | H3 |
| #47 | SSH hardening | H4 |
| #48 | Full NAS security audit | H5 |
| #62 | Rune VM migration runbook | H6 |
| #50 | NFS shared prod/non-prod | H7 |
| #66 | Naming convention | H8 |
| #59 | Deploy NetBox | H9 |
| #58 | OPNsense VM | H1 — deferred, keep as backlog |
| #63 | Deploy DevOps VM | M8 |
| #57 | Ansible collection | M9 |
| #56 | Rename rune-api | L7 |
| #64 | verify_ssl | L1 |
| #30 | Arista EOS mismatch | Existing infra issue |
| #31 | 3com Telnet | Existing infra issue |
| #37 | Proxmox firewall | Existing infra issue |

### Keep but low priority

| Issue | Title | Notes |
|---|---|---|
| #32 | VLAN 620/720 orphaned OSPF | Network cleanup |
| #33 | Vlan60 stale config | Network cleanup |
| #34 | PTP alignment | Network config |
| #38 | Single inter-switch uplink | Phase 2 network |
| #39 | PTP grandmaster SPOF | Phase 2 network |
| #40 | Ansible LXC backup | Infra hygiene |
| #43 | Per-tool sub-projects | Process improvement |
| #45 | Arista 7048T-A | D2 — deferred |
| #49 | noVNC copy/paste | L6 |
| #51 | Ansible LXC DHCP | M6 |

### Close

| Issue | Title | Reason |
|---|---|---|
| #54 | DSM upgrade 7.1.1 → 7.2.x | **Hardware EOL** — DS1513+ max firmware is 7.1.1. No upgrade path. |
| #28 | Module: CCTV | Phase 4c — too early, not actionable |
| #29 | Module: VoIP | Phase 4b — too early, not actionable |

**Summary: Close #54 (EOL), #28, #29 (too early). Keep everything else.**

---

## 6. Summary for @yboujraf

| Decision | Action | Who |
|---|---|---|
| Approve Rune's 3 OS amendments | Apply to OPERATING-STANDARD.md | @yboujraf |
| naming-check.yml | Keep — no changes needed | Done |
| step-ca docs | Remove skeleton | Rune (after approval) |
| lib-synology-dsm: close #33, #38 | Duplicate issues | Rune (after approval) |
| platform-setup: close #54, #28, #29 | EOL / too early | Rune (after approval) |
| All other issues | Keep as backlog | No action |

---

*@yboujraf reviews and approves. Rune executes only what's approved. Opus verifies.*
