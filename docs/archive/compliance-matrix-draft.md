# BY-SYSTEMS Compliance Matrix — First Pass
**Date:** 2026-03-31
**Status:** First pass — covers controls already referenced in ADRs and OPERATING-STANDARD.md only.
**Next pass:** After Phase 2 (Vault) + Phase 3 (Wazuh/CISO Assistant) deployed.

---

## 1. ISO 27001:2022

| Control | Title | BY-SYSTEMS Implementation | Evidence/Artifact | Phase enforced | Status |
|---|---|---|---|---|---|
| A.5.1.2 | Review of policies | OPERATING-STANDARD.md (agent read-only, owner-reviewed) | OPERATING-STANDARD.md | Phase 1 (now) | ⚠️ Partial — no formal review schedule |
| A.8.1.1 | Asset inventory | ADR-0010 (naming convention enables automated inventory) | ADR-0010 | Phase 4 (NetBox) | ⚠️ Partial — naming defined, no inventory tool yet |
| A.9.1.1 | Access control policy | ADR-0010 (group-based RBAC with env scoping) | ADR-0010 | Phase 1 (now) | ✅ Covered |
| A.9.2.1 | User registration/deregistration | ADR-0010 (three account types with defined lifecycle) | ADR-0010 | Phase 1 (now) | ✅ Covered |
| A.9.2.2 | User access provisioning | ADR-0010 (least privilege, permissions enumerated) | ADR-0010 | Phase 1 (now) | ✅ Covered |
| A.9.2.3 | Privileged access management | ADR-0010 (separate `adm_*` accounts, max 2-3 prod admins) | ADR-0010 | Phase 1 (now) | ✅ Covered |
| A.9.2.4 | Management of secret auth info | ADR-0011 (one credential per file, owner tracked, access scoped) | ADR-0011 | Phase 2 (Vault) | ✅ Covered |
| A.9.2.5 | Review of user access rights | ADR-0010 (quarterly review of disabled/expired accounts) | ADR-0010 | Phase 1 (now) | ✅ Covered — process defined, not yet automated |
| A.9.2.6 | Removal of access rights | ADR-0010 (auto-disable on expiry, disable never delete) | ADR-0010 | Phase 1 (now) | ✅ Covered |
| A.9.4.3 | Password management | ADR-0011 (no plaintext in committed files, redaction enforced) | ADR-0011, OPERATING-STANDARD.md §6.1 | Phase 1 (now) | ✅ Covered |
| A.9.4.4 | Use of privileged utility programs | ADR-0010 (break-glass restricted to OOB VLAN) | ADR-0010 | Phase 1 (now) | ✅ Covered |
| A.10.1.1 | Cryptographic controls policy | ADR-0011 (Vault KV v2 for encrypted storage in Phase 2) | ADR-0011, OPERATING-STANDARD.md §6.1/§6.3 | Phase 2 (Vault) | ⚠️ Partial — Phase 1 has no encryption at rest |
| A.12.1.1 | Documented operating procedures | OPERATING-STANDARD.md, lib-synology-dsm ADR-0008 | OPERATING-STANDARD.md | Phase 1 (now) | ✅ Covered |
| A.12.1.4 | Separation of environments | ADR-0010, ADR-0012 (six explicit tiers) | ADR-0010, ADR-0012 | Phase 1 (now) | ✅ Covered |
| A.12.4.1 | Event logging | ADR-0011 (Vault audit trail Phase 2), ADR-0001 (Teleport session recording) | ADR-0001, ADR-0011 | Phase 3 (Wazuh+Teleport) | ⚠️ Partial — Vault and Teleport not yet deployed |
| A.13.1.1 | Network controls | ADR-0010 (standard vs OOB VLAN separation) | ADR-0010 | Phase 1 (now) | ✅ Covered — design defined, enforcement pending (OPNsense) |
| A.13.1.3 | Segregation in networks | ADR-0010 (admin/automation on OOB only) | ADR-0010 | Phase 1 (now) | ✅ Covered — design defined, enforcement pending |
| A.14.2.1 | Secure development policy | lib-synology-dsm ADR-0008, OPERATING-STANDARD.md §5 | OPERATING-STANDARD.md §5, ADR-0008 | Phase 1 (now) | ✅ Covered |
| A.14.2.6 | Secure development environment | ADR-0012 (dev/test isolated from prod) | ADR-0012 | Phase 1 (now) | ✅ Covered |
| A.14.3.1 | Protection of test data | ADR-0012 (prod data anonymized for lower tiers) | ADR-0012 | Phase 1 (now) | ✅ Covered — policy defined, no prod data yet |
| A.18.2.1 | Independent review of information security | Naming draft §10 (revision triggers) | ADR-0010 | Phase 5 (full GRC) | ⚠️ Partial — no formal audit schedule |

---

## 2. NIS2

| Article | Title | BY-SYSTEMS Implementation | Evidence/Artifact | Phase enforced | Status |
|---|---|---|---|---|---|
| Art.21(2)(a) | Risk management measures | ADR-0010, ADR-0012 (env separation), OPERATING-STANDARD.md §7 (RAID) | ADR-0010, ADR-0012, OPERATING-STANDARD.md §7 | Phase 1 (now) | ✅ Covered |
| Art.21(2)(b) | Incident handling | ADR-0010 (automation re-enable with scoped permissions + audit log) | ADR-0010 | Phase 3 (Wazuh+Teleport) | ⚠️ Partial — incident response process not yet formalized |
| Art.21(2)(c) | Business continuity | ADR-0010 (break-glass access as last resort) | ADR-0010 | Phase 3 (Wazuh+Teleport) | ⚠️ Partial — no formal BCP/DRP document |
| Art.21(2)(d) | Supply chain security | ADR-0011 (credential isolation per env) | ADR-0011 | Phase 2 (Vault) | ⚠️ Partial — no vendor risk assessment process |
| Art.21(2)(e) | Security in network/information systems | ADR-0011 (detect-secrets), ADR-0012 (per-env credentials) | ADR-0011, ADR-0012, OPERATING-STANDARD.md §6.1 | Phase 1 (now) | ✅ Covered |
| Art.21(2)(i) | Human resources security | ADR-0010 (account lifecycle tied to employment/contract) | ADR-0010 | Phase 1 (now) | ✅ Covered |

---

## 3. GDPR

| Article | Title | BY-SYSTEMS Implementation | Evidence/Artifact | Phase enforced | Status |
|---|---|---|---|---|---|
| Art.17 | Right to erasure | ADR-0010 (anonymize on explicit request + legal review; default = disabled) | ADR-0010 | Phase 1 (now) | ✅ Covered — policy defined |
| Art.25 | Data protection by design | ADR-0012 (synthetic data in test envs, anonymization for lower tiers) | ADR-0012 | Phase 1 (now) | ✅ Covered |
| Art.32 | Security of processing | ADR-0010, ADR-0012 (env isolation, least privilege, audit trail) | ADR-0010, ADR-0012 | Phase 1 (now) | ✅ Covered |

---

## 4. DORA (Digital Operational Resilience Act)

> First pass — 2026-03-31. Covers controls referenced in current ADRs and OPERATING-STANDARD.md only.

| DORA Pillar | Article | BY-SYSTEMS Implementation | Evidence/Artifact | Status |
|---|---|---|---|---|
| ICT risk management | Art.5-16 | RAID tracking per repo, OPERATING-STANDARD.md §7, ADR-0010 naming enables automated inventory | OPERATING-STANDARD.md §7, ADR-0010, ADR-0013 | ⚠️ Partial — no automated risk register yet |
| ICT incident reporting | Art.17-23 | GitHub Issues + RAID atomic rule (issue+RAID+board = one operation), Discord #releases for comms | OPERATING-STANDARD.md §7.1, GitHub Issues | ⚠️ Partial — manual process, no formal incident response playbook |
| Digital operational resilience testing | Art.24-27 | Unit + integration tests, nox matrix, CI gates on all PRs, 100% coverage gate on lib | OPERATING-STANDARD.md §5, lib-synology-dsm CI | ⚠️ Partial — no TLPT (Threat-Led Penetration Testing) yet |
| ICT third-party risk | Art.28-44 | Secret storage per-env (ADR-0011), provider pinning (Terraform bpg/proxmox v0.99.0), no supply chain policy yet | ADR-0011, ADR-0012 | ⚠️ Planned — supply chain policy (SBOM, dependency audit) not yet in place |
| Information sharing | Art.45-49 | Discord #releases for release comms, git commit history, CHANGELOG per repo | CHANGELOG.md per repo, GitHub releases | ⚠️ Minimal — internal only, no formal threat intelligence sharing |

---

**Status legend:** ✅ Covered | ⚠️ Partial | ❌ Gap
