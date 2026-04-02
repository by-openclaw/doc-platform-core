# ADR-0022: Open source licensing policy

**Status:** Accepted
**Date:** 2026-04-02
**Deciders:** @yboujraf
**Revised:** 2026-04-02 — context corrected: BY-SYSTEMS is a systems integrator, not a SaaS vendor.

---

## Context

BY-SYSTEMS is a **systems integrator** operating its own internal platform infrastructure and hosting select services for its own operational purposes. It does not redistribute software as SaaS, does not package or resell open-source tools to customers, and does not offer software as a service to third parties.

The licensing concern is therefore:
- **Can we legally deploy and operate this tool internally?**
- **Can we operate it in an environment that serves our internal teams and infrastructure?**

Not: "can we redistribute it?" or "does this obligate us to publish source?"

Despite this constrained scope, licensing still matters:
- Redis relicensed to SSPL from v7.4+ — SSPL's definition of "Service" is broad; internal use assessment required before upgrade
- HashiCorp Vault uses BUSL-1.1 — "production use" restriction applies until 4-year conversion; internal production deployment requires review
- No platform-wide license tracking exists — tools are adopted without a license record on file

This ADR establishes the platform licensing policy. The governing standard is `docs/standards/licensing-standard.md`.

## Decision

**All tools must have a documented SPDX license identifier.** No tool may be deployed without a license record on file.

**Approved licenses** (no further review required for internal deployment and operation):
Apache-2.0, MIT, GPL-2.0, GPL-3.0, LGPL-2.1, LGPL-3.0, MPL-2.0, BSD-2-Clause, BSD-3-Clause.

**Flagged licenses** (written approval from @yboujraf required before adoption or upgrade):
BUSL-1.1, SSPL-1.0, AGPL-3.0, any commercial or proprietary license.

Assessment basis for flagged licenses: **internal deployment and operation only** — redistribution and SaaS clauses do not apply to BY-SYSTEMS's use model.

**Redis license flag:** Redis ≥ 7.4 uses SSPL-1.0. No version pinning — latest version is used. SSPL permits internal deployment and operation; no redistribution or SaaS offering is involved. License terms are respected as-is.

**Vault license flag:** HashiCorp Vault uses BUSL-1.1. No version pinning. Internal production deployment is within BUSL-1.1 terms for an integrator operating its own infrastructure. License terms are respected as-is.

**Review process:** Self-assessment by @yboujraf sufficient for internal-use flagged licenses. No external legal review required unless a tool is being evaluated for customer-facing deployment.

**Bill of Materials (BoM):** `docs/2026-04-01-infra-bom-poc-proxmox.md` is the current platform BoM. SPDX license field is mandatory in all tool `docs/licensing.md` files and in the BoM.

## CISO mapping

### ISO/IEC 27001:2022

| Control | Title | Status | Notes |
|---|---|---|---|
| A.5.20 | Addressing information security within supplier agreements | ✓ Covered | License policy defined; all tools auditable OSS |
| A.8.30 | Outsourced development | ✓ Covered | Source available for all deployed tools; license tracked per tool |

### NIS2 (Directive 2022/2555)

| Article | Requirement | Status | Notes |
|---|---|---|---|
| Art. 21(2)(d) | Supply chain security | ✓ Covered | License review required per tool before deployment; BoM maintained |

### GDPR (Regulation 2016/679)

Not applicable — this ADR does not touch personal data processing.

## Licensing

| Tool / Service | SPDX license | Status | Notes |
|---|---|---|---|
| Redis (any version) | BSD-3-Clause / SSPL-1.0 | ✅ Accepted | SSPL permits internal deployment; no SaaS/redistribution involved |
| HashiCorp Vault | BUSL-1.1 | ✅ Accepted | Internal production use within BUSL terms; integrator context |
| OpenTofu | MPL-2.0 | ✅ Approved | OSS fork of Terraform; use if Vault BUSL concern grows |

## Consequences

**Enables:**
- Legal protection — no unknowing deployment of license-restricted tools
- Consistent license documentation across all platform tools
- Audit trail for future compliance (ISO 27001, customer audits)

**Constrains:**
- Every new tool adoption requires a license check before PR merge
- Flagged licenses (BUSL/SSPL/AGPL) require a documented internal-use assessment — not a block, but must be on record

**Known risks:**
- BoM license field completeness depends on per-tool `docs/licensing.md` being filled in — not yet complete for all 29 tools

## References

- `docs/standards/licensing-standard.md` — platform licensing standard
- `docs/2026-04-01-infra-bom-poc-proxmox.md` — platform BoM
- [SPDX License List](https://spdx.org/licenses/)
- [Redis license change (v7.4)](https://redis.io/blog/redis-adopts-dual-source-available-licensing/)
- [HashiCorp BUSL announcement](https://www.hashicorp.com/blog/hashicorp-adopts-business-source-license)
