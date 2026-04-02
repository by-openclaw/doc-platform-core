<!--
  SCOPE GUARD — INFRA ADR
  ========================
  This template is for infrastructure decisions only.
  DO NOT use this template for:
  - Library implementation details (DI, test strategy, log format)
  - Developer tooling (devcontainer, pre-commit hooks)
  - App-level credential handling
  Wrong template = PR blocked.
-->

# ADR-0022: Open source licensing policy

**Status:** Draft
**Date:** 2026-04-02
**Deciders:** @yboujraf

---

## Context

The BY-SYSTEMS platform is built exclusively on open-source tools. As the platform matures and potentially serves commercial customers, licensing compliance becomes a legal and operational risk:

- Redis relicensed to SSPL from v7.4+ — upgrading without review could create licensing obligations
- HashiCorp Vault uses BUSL-1.1 with a 4-year conversion clause — commercial deployment requires review
- AGPL v3 tools require source disclosure if offered as a network service
- No platform-wide approved license list exists — individual tools are adopted without licence checking

This ADR establishes the platform licensing policy. The governing standard is `docs/standards/licensing-standard.md`.

## Decision

**All tools must have a documented SPDX license identifier.** No tool may be deployed without a license assessment on record.

**Approved licenses** (no further review required): Apache-2.0, MIT, GPL-2.0, GPL-3.0, LGPL-2.1, LGPL-3.0, MPL-2.0, BSD-2-Clause, BSD-3-Clause.

**Flagged licenses** (written approval from @yboujraf required before adoption or upgrade): BUSL-1.1, SSPL-1.0, AGPL-3.0, any commercial/proprietary license.

**Redis license flag:** Redis ≥ 7.4 uses SSPL-1.0. The platform currently pins Redis below 7.4. Any upgrade to 7.4+ requires @yboujraf sign-off and a documented license review.

**Vault license flag:** HashiCorp Vault uses BUSL-1.1. The 4-year conversion clause means the license converts to an OSS license [OWNER TO DEFINE: review conversion date for current Vault version]. Commercial deployment requires @yboujraf sign-off.

**Review process:** [OWNER TO DEFINE: review process steps — e.g., legal review required for BUSL/SSPL, or self-assessment sufficient?]

**Bill of Materials (BoM):** [OWNER TO DEFINE: BoM location and update cadence — e.g., `docs/bom.md`, updated each platform release]

## VM / Resource Spec

Not applicable.

## Network

Not applicable.

## Storage

Not applicable.

## TLS / PKI

Not applicable.

## CISO mapping

> Applies only to controls directly relevant to this ADR's scope.

### ISO/IEC 27001:2022

| Control | Title | Status | Notes |
|---|---|---|---|
| A.5.20 | Addressing information security within supplier agreements | ⚠ Partial | License policy defined; formal supplier assessment process not yet in place |
| A.8.30 | Outsourced development | ✓ Covered | All tooling is auditable OSS — source available for review |

### NIS2 (Directive 2022/2555)

| Article | Requirement | Status | Notes |
|---|---|---|---|
| Art. 21(2)(d) | Supply chain security | ⚠ Partial | License review process defined as placeholder; BoM location pending |

### GDPR (Regulation 2016/679)

Not applicable — this ADR does not touch personal data processing.

## Licensing

| Tool / Service | SPDX license | Tier | License ref |
|---|---|---|---|
| Redis (current, < 7.4) | BSD-3-Clause | Free | https://github.com/redis/redis/blob/7.2/LICENSE.txt |
| Redis (≥ 7.4) | SSPL-1.0 | ⚠ FLAGGED | https://github.com/redis/redis/blob/unstable/LICENSE.txt |
| HashiCorp Vault | BUSL-1.1 | ⚠ FLAGGED | https://github.com/hashicorp/vault/blob/main/LICENSE |

## Consequences

**Enables:**
- Legal protection — no unknowing use of license-restricted tools in commercial deployments
- Consistent license documentation across all 29+ platform tools
- Audit trail for future compliance (ISO 27001, customer audits)

**Constrains:**
- Redis must not be upgraded to ≥ 7.4 without sign-off
- Vault commercial deployment requires sign-off
- Every new tool adoption requires a license check before PR merge

**Known risks:**
- BoM location and update process not yet defined — license tracking is manual until a BoM tool is adopted
- Review process for flagged licenses is placeholder — legal exposure if a flagged tool is upgraded without process
- AGPL tools not currently in platform but could be introduced without this policy — policy must be communicated to all contributors

## References

- `docs/standards/licensing-standard.md` — platform licensing standard (governing document)
- [SPDX License List](https://spdx.org/licenses/)
- [Redis license change (v7.4)](https://redis.io/blog/redis-adopts-dual-source-available-licensing/)
- [HashiCorp BUSL announcement](https://www.hashicorp.com/blog/hashicorp-adopts-business-source-license)
- [SSPL analysis — Open Source Initiative](https://opensource.org/blog/the-sspl-is-not-an-open-source-license)
