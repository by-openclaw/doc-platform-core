# ADR-0006 — Platform Charter

**Date:** 2026-03-28
**Status:** Accepted — gates all future work
**Scope:** All BY-SYSTEMS PoC platform work, all repos, all agents
**Deciders:** @yboujraf
**Revised:** 2026-04-02 — stripped to single-scope (layer model + gating rules). All other content moved to authoritative ADRs.

> **Rule #1:** Nothing proceeds to the next layer until the current layer is documented, tested, and signed off.
> **Rule #2:** Every decision lives in an ADR. Every finding lives in RAID + GitHub Issue + Project board.
> **Rule #3:** AI agents assist. They do not decide. They do not skip layers.

---

## Context

The platform requires a strict build order. Deploying services before the foundation is stable creates cascading failures, untraceable dependencies, and security gaps. A formal layer model was needed to govern sequencing and acceptance criteria.

## Decision

The platform is built in strict sequential layers. No layer may begin until the previous layer passes its acceptance criteria, is documented, and is signed off by @yboujraf.

```
┌─────────────────────────────────────────────────────────────────┐
│  Layer 0 — Standards & Templates                                │
│  Define once. Apply forever. No exceptions.                      │
│  Acceptance: ADRs written, standards files exist, templates set  │
├─────────────────────────────────────────────────────────────────┤
│  Layer 1 — Proxmox Base                                         │
│  Cloud-init template finalized. VM/LXC baseline locked.         │
│  Acceptance: any VM provisioned = same baseline, zero manual     │
├─────────────────────────────────────────────────────────────────┤
│  Layer 2 — Vault                                                │
│  Secrets management. Nothing deploys credentials without Vault. │
│  Acceptance: Vault cluster up, PKI + KV mounted, policies set   │
├─────────────────────────────────────────────────────────────────┤
│  Layer 3 — Identity (Authentik + EntraID OIDC)                  │
│  All platform users from Vault. SSH via CA. No static keys.     │
│  Acceptance: all users from Vault, SSH via CA, no static keys   │
├─────────────────────────────────────────────────────────────────┤
│  Layer 4 — Storage (Synology)                                   │
│  NFS shares, CRUD API, permissions via lib-synology-dsm.        │
│  Acceptance: lib tested, CI green, shares provisioned via code  │
├─────────────────────────────────────────────────────────────────┤
│  Layer 5 — Platform Services                                    │
│  NetBox, GitLab CE, monitoring, etc.                            │
│  Acceptance: each service has runbook, CI gate, RAID entry      │
└─────────────────────────────────────────────────────────────────┘
```

**Parallel development exception:** Layer 4 library work (`lib-synology-dsm`) is developed in parallel with infrastructure layers — it has no infra runtime dependency.

## Gating rules

- A layer is "done" when: documented + tested + signed off by @yboujraf. Not before.
- Any agent or contributor detecting a gap in a lower layer must stop and raise a RAID item before proceeding.
- No service in Layer 5 may be deployed before Layers 1–4 are complete.

## Consequences

- Work is sequential. This trades speed for correctness and auditability.
- Parallel lib/tooling work is permitted as an explicit exception.
- Every layer transition requires a sign-off — not assumed, not implicit.

## Superseded content

The original ADR-0006 was written as a bootstrap document before proper ADRs existed. All sections beyond the layer model have been moved to their authoritative locations:

| Original section | Now in |
|---|---|
| Network & IP plan | ADR-0015 — Network VLAN Architecture |
| VM user standard | ADR-0021 — Hardening Baseline |
| Repository standard | ADR-0002 — Repository Structure and Naming |
| RAID tracking rules | ADR-0003 — Issue Tracking Standard |
| CLAUDE.md standard | Per-repo AGENTS.md + CONTRIBUTING.md |
| Sub-agent rules | Workspace AGENTS.md |
| AI model routing | Workspace TOOLS.md |
| Diagram standard | CONTRIBUTING.md |
| Contribution flow | CONTRIBUTING.md |
| Open actions (RAID) | RAID.md + GitHub Issues |

> Archive of original: `docs/adr/archive/0006-platform-charter.v0.md`

---

## CISO mapping

### ISO/IEC 27001:2022

| Control | Title | Status | Notes |
|---|---|---|---|
| A.5.1 | Policies for information security | ✓ Covered | Charter defines governing rules and layer sequencing |
| A.8.32 | Change management | ✓ Covered | No layer advance without documentation, test, and sign-off |
| A.8.9 | Configuration management | ✓ Covered | Every layer transition is gated and documented |

### NIS2 (Directive 2022/2555)

| Article | Requirement | Status | Notes |
|---|---|---|---|
| Art. 21(2)(a) | Risk management policies | ✓ Covered | Layer gating and mandatory documentation reduce uncontrolled change risk |

### GDPR (Regulation 2016/679)

Not applicable — this ADR governs platform build sequencing only.

---

## References

- ADR-0001 — Platform Stack Decisions
- ADR-0002 — Repository Structure and Naming
- ADR-0003 — Issue Tracking Standard
- ADR-0015 — Network VLAN Architecture
- ADR-0021 — Hardening Baseline
- `docs/raid.md` — RAID tracker
- `CONTRIBUTING.md` — contribution flow, commit standard, diagram standard
