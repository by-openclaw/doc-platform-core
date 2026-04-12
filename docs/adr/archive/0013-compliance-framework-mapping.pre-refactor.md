# ADR-0013: Compliance Framework Mapping (First Pass)

**Status:** Accepted
**Date:** 2026-03-31
**Deciders:** @yboujraf

## Context

BY-SYSTEMS targets four compliance frameworks (OPERATING-STANDARD.md §6.4): ISO 27001:2022, NIS2, GDPR, and DORA. Controls are referenced across ADRs and operational documents, but there is no single mapping that shows which controls are covered, which are partially addressed, and which are gaps.

This first pass maps only controls **already referenced** in existing ADRs and OPERATING-STANDARD.md. It does not attempt to map all 93 ISO 27001 Annex A controls or all NIS2 articles. Gaps are flagged for future passes.

## Decision

Maintain a compliance framework mapping as a living ADR. Each control is linked to the BY-SYSTEMS implementation (ADR, policy section, or tool) that addresses it. Controls not yet addressed are flagged as gaps.

### ISO 27001:2022 Annex A — Controls Currently Referenced

| Control | Title | BY-SYSTEMS Implementation | Status |
|---|---|---|---|
| A.5.1.2 | Review of policies | OPERATING-STANDARD.md (agent read-only, owner-reviewed) | Partial — no formal review schedule |
| A.8.1.1 | Asset inventory | Naming convention enables automated inventory; asset names encode env and function | Partial — naming defined, no inventory tool yet (NetBox Phase 5) |
| A.9.1.1 | Access control policy | Group-based RBAC with env scoping: `grp-{env}-{role}` — defined in naming & identity convention | Covered |
| A.9.2.1 | User registration/deregistration | Three account types (human/service/temp) with defined lifecycle — defined in naming & identity convention | Covered |
| A.9.2.2 | User access provisioning | Least privilege; permissions enumerated per service account — defined in naming & identity convention | Covered |
| A.9.2.3 | Privileged access management | Separate `adm_*` accounts, max 2-3 prod admins, quarterly review | Covered |
| A.9.2.4 | Management of secret auth info | One credential per JSON file, owner tracked, access scoped — pre-Vault Phase 1 format | Covered |
| A.9.2.5 | Review of user access rights | Quarterly review of disabled/expired accounts — policy defined, not yet automated | Covered — process defined, not yet automated |
| A.9.2.6 | Removal of access rights | Auto-disable on expiry, disable-never-delete policy | Covered |
| A.9.4.3 | Password management | No plaintext in committed files; redaction enforced; detect-secrets pre-commit hooks | Covered |
| A.9.4.4 | Use of privileged utility programs | Break-glass restricted to OOB VLAN; standard users cannot reach OOB VLAN | Covered |
| A.10.1.1 | Cryptographic controls policy | Vault KV v2 for encrypted storage in Phase 2; Phase 1 JSON files are not encrypted at rest | Partial — Phase 1 has no encryption at rest |
| A.12.1.1 | Documented operating procedures | OPERATING-STANDARD.md defines agent and operator procedures | Covered |
| A.12.1.4 | Separation of environments | Six explicit environment tiers (poc/dev/test/staging/acc/prod) with promotion gates | Covered |
| A.12.4.1 | Event logging | Vault audit trail (Phase 2); Teleport session recording (Phase 3) | Partial — Vault and Teleport not yet deployed |
| A.13.1.1 | Network controls | Standard vs OOB VLAN separation; network-layer block on standard users reaching OOB | Covered — design defined, enforcement pending OPNsense deployment |
| A.13.1.3 | Segregation in networks | Admin/automation on OOB only; SVC/DMZ/MGMT VLANs defined | Covered — design defined, enforcement pending |
| A.14.2.1 | Secure development policy | Consistent API pattern, ensure/idempotent standard, OPERATING-STANDARD.md §5 | Covered |
| A.14.2.6 | Secure development environment | Dev/test environments isolated from prod via six-tier pipeline | Covered |
| A.14.3.1 | Protection of test data | Prod data anonymized before use in staging/acc tiers | Covered — policy defined, no prod data yet |
| A.18.2.1 | Independent review of information security | OPERATING-STANDARD.md defines revision triggers | Partial — no formal audit schedule |

### NIS2 Directive — Articles Currently Referenced

| Article | Title | BY-SYSTEMS Implementation | Status |
|---|---|---|---|
| Art.21(2)(a) | Risk management measures | Environment separation + RAID log in docs/raid.md + OPERATING-STANDARD.md §7 | Covered |
| Art.21(2)(b) | Incident handling | Scoped automation accounts with env isolation; audit log from naming convention | Partial — incident response process not yet formalized |
| Art.21(2)(c) | Business continuity | Break-glass access as last resort; platform charter defines work layers | Partial — no formal BCP/DRP document |
| Art.21(2)(d) | Supply chain security | Credential isolation per env prevents cross-env blast radius | Partial — no vendor risk assessment process |
| Art.21(2)(e) | Security in network/information systems | detect-secrets pre-commit hooks; per-env credentials and certificates | Covered |
| Art.21(2)(i) | Human resources security | Account lifecycle tied to employment/contract; disable-never-delete policy | Covered |

### GDPR — Articles Currently Referenced

| Article | Title | BY-SYSTEMS Implementation | Status |
|---|---|---|---|
| Art.17 | Right to erasure | Anonymise on explicit request + legal review; default = disabled (audit trail preserved) | Covered — policy defined |
| Art.25 | Data protection by design | Test environments use synthetic data; prod data anonymized for lower tiers | Covered |
| Art.32 | Security of processing | Environment isolation, least privilege, audit trail via account lifecycle | Covered |

### DORA — Status

DORA (Digital Operational Resilience Act) is listed as a compliance target in OPERATING-STANDARD.md §6.4. **No DORA-specific controls are referenced in any current ADR.** DORA primarily applies to financial entities and their ICT service providers.

**Gap:** DORA mapping requires:
- ICT risk management framework (Art.5-16)
- ICT-related incident reporting (Art.17-23)
- Digital operational resilience testing (Art.24-27)
- ICT third-party risk management (Art.28-44)

These overlap significantly with ISO 27001 and NIS2 controls already in place. A dedicated DORA pass should map existing controls to DORA articles and identify net-new gaps.

### Identified Gaps (Not Yet Covered)

| Gap | Framework | Priority | Notes |
|---|---|---|---|
| No formal policy review schedule | ISO A.5.1.2 | MEDIUM | Define annual review cadence |
| No asset inventory tool | ISO A.8.1.1 | HIGH | NetBox (Phase 5) will address this |
| No encryption at rest for Phase 1 secrets | ISO A.10.1.1 | HIGH | Vault (Phase 2) will address this |
| No centralized event logging | ISO A.12.4.1 | HIGH | Wazuh SIEM (Phase 3) will address this |
| Network segmentation not yet enforced | ISO A.13.1.1/A.13.1.3 | HIGH | OPNsense deployment pending |
| No formal incident response process | NIS2 Art.21(2)(b) | HIGH | Define IRP document |
| No BCP/DRP document | NIS2 Art.21(2)(c) | MEDIUM | Create after core infra stable |
| No vendor risk assessment | NIS2 Art.21(2)(d) | LOW | Relevant when external vendors onboarded |
| No DORA mapping | DORA | LOW | Overlap with ISO/NIS2 — map after those are complete |
| No formal audit schedule | ISO A.18.2.1 | MEDIUM | Define after GRC tool (CISO Assistant) deployed |

## Consequences

**Positive:**
- Single document shows compliance posture at a glance — auditor-friendly
- Gaps are explicit and prioritized — no false sense of compliance
- Each control links to its implementation (ADR or policy section) — evidence chain is traceable
- Future passes add controls incrementally — manageable scope

**Negative:**
- First pass is incomplete — does not cover all 93 ISO Annex A controls (intentional)
- Mapping is manual — no automated compliance checking until CISO Assistant deployed
- Controls marked "Covered" are policy-level only — operational enforcement depends on Phase 2-5 tooling (Vault, Authentik, Wazuh, NetBox)

## CISO mapping

> This ADR IS the compliance mapping registry for the platform. The section below summarises its own governance posture.

### ISO/IEC 27001:2022

| Control | Title | Status | Notes |
|---|---|---|---|
| A.5.1 | Policies for information security | ⚠ Partial | Platform policies documented in OPERATING-STANDARD.md; formal review schedule not yet defined |

### NIS2 (Directive 2022/2555)

| Article | Requirement | Status | Notes |
|---|---|---|---|
| Art. 21(2)(a) | Risk management policies | ✓ Covered | This ADR is the compliance evidence register |

### GDPR (Regulation 2016/679)

Not applicable to this meta-ADR directly.

## Notes

- **FIRST PASS ONLY** — covers controls already referenced in existing platform decisions and OPERATING-STANDARD.md
- Does NOT invent coverage — if a control is not referenced in an existing document, it is flagged as a gap
- GRC tool: CISO Assistant (open source) — when deployed, this mapping will be imported as the baseline
- Next pass: after Phase 2 (Vault) and Phase 3 (observability/security) tooling is deployed
- DORA pass: schedule after ISO 27001 and NIS2 mappings are complete (significant overlap expected)
- Platform stack decisions reference Teleport (session recording) and Wazuh (SIEM) as Phase 3 deliverables — controls dependent on these are marked Partial
