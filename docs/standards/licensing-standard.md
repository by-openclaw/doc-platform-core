<!--
| Field        | Value            |
|--------------|------------------|
| Created      | 2026-04-02       |
| Last updated | 2026-04-02       |
| Updated by   | Opus             |
-->

# Licensing Standard

**Status:** Draft
**Date:** 2026-04-02
**Owner:** @yboujraf
**ADR:** ADR-0022 (stub — pending @yboujraf sign-off)

## Rule

All platform tools must use an approved open-source license. Flagged licenses (BUSL, SSPL, AGPL) require explicit review and written approval from @yboujraf before adoption or upgrade. No tool may be deployed without a documented SPDX license identifier.

## Approved licenses

| SPDX identifier | License name | Notes |
|---|---|---|
| Apache-2.0 | Apache License 2.0 | Approved |
| MIT | MIT License | Approved |
| GPL-2.0 | GNU GPL v2 | Approved |
| GPL-3.0 | GNU GPL v3 | Approved |
| LGPL-2.1 | GNU Lesser GPL v2.1 | Approved |
| LGPL-3.0 | GNU Lesser GPL v3 | Approved |
| MPL-2.0 | Mozilla Public License 2.0 | Approved |
| BSD-2-Clause | BSD 2-Clause | Approved |
| BSD-3-Clause | BSD 3-Clause | Approved |

## Flagged licenses (review required)

| SPDX identifier | License name | Risk | Action required |
|---|---|---|---|
| BUSL-1.1 | Business Source License 1.1 | Converts to OSS after [OWNER TO DEFINE: N] years; commercial use restricted until then | Review per tool — written approval required from @yboujraf |
| SSPL-1.0 | Server Side Public License v1 | Copyleft extends to entire service stack if you offer as SaaS | Review per tool — written approval required |
| AGPL-3.0 | GNU Affero GPL v3 | Network copyleft — modifying and running as a service requires source disclosure | Review per tool — written approval required |
| Commercial | Proprietary / paid tier | Cost, vendor lock-in | Must be explicitly approved with cost justification |

## Known flagged tools in platform

| Tool | License | Version flagged | Status |
|---|---|---|---|
| Redis | SSPL-1.0 (v7.4+) | ≥ 7.4 | ⚠ FLAGGED — verify version before upgrade. SSPL applies only from v7.4+ |
| HashiCorp Vault | BUSL-1.1 | Current | ⚠ FLAGGED — BUSL with 4-year conversion clause. Review before commercial deployment |

## Requirements

1. Every tool MUST have a documented SPDX license identifier in its `docs/licensing.md`.
2. Approved licenses may be adopted without further review.
3. Flagged licenses MUST NOT be adopted or upgraded to without written approval from @yboujraf.
4. The review process for flagged licenses: [OWNER TO DEFINE: review process steps — e.g., legal review, risk sign-off].
5. License changes (e.g., Redis < 7.4 → ≥ 7.4) MUST be caught at upgrade time and escalated before deployment.
6. The platform maintains a Bill of Materials (BoM) listing all tools, versions, and licenses: [OWNER TO DEFINE: BoM location and update cadence].

## Compliance table

| Requirement | Test | Pass condition |
|---|---|---|
| SPDX identifier documented | Check `docs/licensing.md` per tool | SPDX field populated |
| No unapproved flagged licenses | Audit all tools against flagged list | Zero flagged licenses without written approval |
| Redis version check | `docker inspect redis | grep Version` | Version < 7.4 OR flagged approval on file |
| Vault BUSL review | Check approval record | Written approval from @yboujraf on file |

## Override procedure

To override this standard for a specific tool:
1. Create `tools/{tool}/docs/override-licensing.md`
2. State: What is different / Why / Compensating control / Reviewed by @yboujraf
3. PR must include override doc before merge is allowed

## References

- [SPDX License List](https://spdx.org/licenses/)
- [Business Source License (BUSL-1.1)](https://mariadb.com/bsl11/)
- [Server Side Public License (SSPL)](https://www.mongodb.com/licensing/server-side-public-license)
- [AGPL v3](https://www.gnu.org/licenses/agpl-3.0.html)
- [Redis license change announcement (v7.4)](https://redis.io/blog/redis-adopts-dual-source-available-licensing/)
- [HashiCorp BUSL announcement](https://www.hashicorp.com/blog/hashicorp-adopts-business-source-license)
