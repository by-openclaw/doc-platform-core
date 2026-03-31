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
| A.8.1.1 | Asset inventory | ADR-0010 (naming convention enables automated inventory) | Partial — naming defined, no inventory tool yet (NetBox Phase 5) |
| A.9.1.1 | Access control policy | ADR-0010 (group-based RBAC with env scoping) | Covered |
| A.9.2.1 | User registration/deregistration | ADR-0010 (three account types with defined lifecycle) | Covered |
| A.9.2.2 | User access provisioning | ADR-0010 (least privilege, permissions enumerated) | Covered |
| A.9.2.3 | Privileged access management | ADR-0010 (separate `adm_*` accounts, max 2-3 prod admins) | Covered |
| A.9.2.4 | Management of secret auth info | ADR-0011 (one credential per file, owner tracked, access scoped) | Covered |
| A.9.2.5 | Review of user access rights | ADR-0010 (quarterly review of disabled/expired accounts) | Covered — process defined, not yet automated |
| A.9.2.6 | Removal of access rights | ADR-0010 (auto-disable on expiry, disable never delete) | Covered |
| A.9.4.3 | Password management | ADR-0011 (no plaintext in committed files, redaction enforced) | Covered |
| A.9.4.4 | Use of privileged utility programs | ADR-0010 (break-glass restricted to OOB VLAN) | Covered |
| A.10.1.1 | Cryptographic controls policy | ADR-0011 (Vault KV v2 for encrypted storage in Phase 2) | Partial — Phase 1 has no encryption at rest |
| A.12.1.1 | Documented operating procedures | OPERATING-STANDARD.md, lib-synology-dsm ADR-0008 | Covered |
| A.12.1.4 | Separation of environments | ADR-0010, ADR-0012 (six explicit tiers) | Covered |
| A.12.4.1 | Event logging | ADR-0011 (Vault audit trail Phase 2), ADR-0001 (Teleport session recording) | Partial — Vault and Teleport not yet deployed |
| A.13.1.1 | Network controls | ADR-0010 (standard vs OOB VLAN separation) | Covered — design defined, enforcement pending (OPNsense) |
| A.13.1.3 | Segregation in networks | ADR-0010 (admin/automation on OOB only) | Covered — design defined, enforcement pending |
| A.14.2.1 | Secure development policy | lib-synology-dsm ADR-0008 (consistent API), OPERATING-STANDARD.md §5 | Covered |
| A.14.2.6 | Secure development environment | ADR-0012 (dev/test isolated from prod) | Covered |
| A.14.3.1 | Protection of test data | ADR-0012 (prod data anonymized for lower tiers) | Covered — policy defined, no prod data yet |
| A.18.2.1 | Independent review of information security | Naming draft §10 (revision triggers) | Partial — no formal audit schedule |

### NIS2 Directive — Articles Currently Referenced

| Article | Title | BY-SYSTEMS Implementation | Status |
|---|---|---|---|
| Art.21(2)(a) | Risk management measures | ADR-0010, ADR-0012 (env separation), OPERATING-STANDARD.md §7 (RAID) | Covered |
| Art.21(2)(b) | Incident handling | ADR-0010 (automation re-enable with scoped permissions + audit log) | Partial — incident response process not yet formalized |
| Art.21(2)(c) | Business continuity | ADR-0010 (break-glass access as last resort) | Partial — no formal BCP/DRP document |
| Art.21(2)(d) | Supply chain security | ADR-0011 (credential isolation per env) | Partial — no vendor risk assessment process |
| Art.21(2)(e) | Security in network/information systems | ADR-0011 (detect-secrets), ADR-0012 (per-env credentials) | Covered |
| Art.21(2)(i) | Human resources security | ADR-0010 (account lifecycle tied to employment/contract) | Covered |

### GDPR — Articles Currently Referenced

| Article | Title | BY-SYSTEMS Implementation | Status |
|---|---|---|---|
| Art.17 | Right to erasure | ADR-0010 (anonymize on explicit request + legal review; default = disabled) | Covered — policy defined |
| Art.25 | Data protection by design | ADR-0012 (synthetic data in test envs, anonymization for lower tiers) | Covered |
| Art.32 | Security of processing | ADR-0010, ADR-0012 (env isolation, least privilege, audit trail) | Covered |

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

## Compliance

This ADR **is** the compliance mapping. It references all four target frameworks:
- **ISO 27001:2022** — 21 Annex A controls mapped
- **NIS2** — 6 Art.21(2) sub-articles mapped
- **GDPR** — 3 articles mapped
- **DORA** — flagged as gap, pending dedicated pass

## Notes

- **FIRST PASS ONLY** — covers controls already referenced in existing ADRs and OPERATING-STANDARD.md
- Does NOT invent coverage — if a control is not referenced in an existing document, it is flagged as a gap
- GRC tool: CISO Assistant (open source) — when deployed, this mapping will be imported as the baseline
- Next pass: after Phase 2 (Vault) and Phase 3 (observability/security) tooling is deployed
- DORA pass: schedule after ISO 27001 and NIS2 mappings are complete (significant overlap expected)
- ADR-0001 (platform stack) references Teleport and Wazuh for audit/SIEM — these are Phase 3 deliverables
