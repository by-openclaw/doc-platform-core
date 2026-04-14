# security/0005 — Open Source Licensing Policy

**Status:** Draft
**Date:** 2026-04-14 (supersedes flat ADR-0022, 2026-04-02)
**Scope:** Licensing rules for all tools and libraries deployed on the platform — approved SPDX license list, flagged licenses requiring assessment, internal-use-only model, BoM tracking requirements. Does **not** define tool selection or platform stack inventory.
**Related:** `security/0002-compliance-mapping`, `infra/0001-platform-stack`, `lib/python/0001-design-standard`

---

## Context

BY-SYSTEMS is a **systems integrator** operating its own internal platform infrastructure and hosting services for its own operational purposes. It does **not** redistribute software as SaaS, does **not** resell open-source tools to third parties, and does **not** offer software as a service to external customers.

The licensing question is therefore:
- **Can we legally deploy and operate this tool internally?**

Not:
- "Can we redistribute it?"
- "Does this obligate us to publish our own source?"
- "Does hosting this tool for our customers trigger the SaaS clauses?"

Despite this narrower scope, licensing still matters because:

- **Redis** relicensed to **SSPL-1.0** from v7.4+ — SSPL's definition of "Service" is broad; internal-use assessment required before upgrade
- **HashiCorp Vault** uses **BUSL-1.1** — "production use" restriction until 4-year conversion to MPL; internal production deployment requires review
- Without a platform-wide license tracking policy, tools are adopted ad-hoc without a license record on file — incompatible with ISO 27001:2022 A.5.20 and supply-chain audit requirements

## Decision

### 1. Internal-use-only assessment model

**All licensing assessments on this platform apply to internal deployment and operation only.** Redistribution and SaaS clauses in any license (SSPL, BUSL, AGPL, commercial) are not triggered because BY-SYSTEMS does not redistribute or offer SaaS.

When a license is assessed, the question is:
- Does this license permit us to install, configure, operate, and expose this tool to our own internal team on our own infrastructure?

If **yes** → the tool may be deployed.
If **no** → the tool is blocked.
If **uncertain** → written approval from @yboujraf before deployment or upgrade (see §3 Flagged licenses).

### 2. Approved SPDX licenses — no further review required

The following SPDX license identifiers are **approved for internal deployment** without additional review:

| Category | SPDX IDs |
|---|---|
| Permissive | `MIT`, `Apache-2.0`, `BSD-2-Clause`, `BSD-3-Clause`, `ISC`, `Zlib` |
| Weak copyleft | `LGPL-2.1`, `LGPL-2.1-only`, `LGPL-2.1-or-later`, `LGPL-3.0`, `LGPL-3.0-only`, `LGPL-3.0-or-later`, `MPL-2.0` |
| Strong copyleft (non-SaaS) | `GPL-2.0`, `GPL-2.0-only`, `GPL-2.0-or-later`, `GPL-3.0`, `GPL-3.0-only`, `GPL-3.0-or-later` |

**Rules:**
- Any tool whose license is in this list can be deployed without written approval — the license record (SPDX ID + source URL) still must be captured in the tool's `docs/licensing.md` file
- `GPL-*` is approved because BY-SYSTEMS does not distribute the tool to third parties; the copyleft obligation triggers on distribution, not on internal use

### 3. Flagged licenses — written approval required before adoption

The following SPDX identifiers require **written approval from @yboujraf** before the tool is adopted, upgraded, or touched at all:

| SPDX ID | License | Why flagged |
|---|---|---|
| `BUSL-1.1` | Business Source License 1.1 | "Production use" restriction until conversion date; internal-use assessment required |
| `SSPL-1.0` | Server Side Public License 1.0 | Broad "Service" definition; internal-use assessment required |
| `AGPL-3.0`, `AGPL-3.0-only`, `AGPL-3.0-or-later` | GNU Affero GPL v3 | "Remote network interaction" copyleft trigger; internal-use assessment required |
| `Commons Clause` | Commons Clause addendum | Commercial-use restriction; internal-use assessment required |
| `Elastic-2.0`, `ELv2` | Elastic License v2 | Restricts hosted SaaS, managed service, "circumvention" clauses |
| `Proprietary`, `Commercial`, any non-OSI license | — | Not open source; separate approval process applies |

**Assessment process:**
1. Open an issue titled `license-assessment: {tool}` in `doc-platform-core`
2. Attach the license text, the tool's `README` / `LICENSE` file content, and the use case
3. @yboujraf writes a short assessment in the issue: "approved for internal use because …" or "blocked because …"
4. On approval: the tool is added to `docs/licensing.md` with SPDX ID, assessment issue link, and date
5. Assessment decisions are tracked in `security/0002-compliance-mapping` as inputs to ciso-assistant

**Self-assessment is sufficient** for internal-use decisions by @yboujraf — no external legal review required unless a tool is being evaluated for customer-facing deployment (out of scope for this ADR).

### 4. Flagged license decisions (current)

The following flagged licenses have been **assessed and approved for internal deployment** on BY-SYSTEMS infrastructure:

| Tool | SPDX ID | Status | Assessment |
|---|---|---|---|
| **Redis** (≥ 7.4) | `SSPL-1.0` (dual-licensed with `RSAL-v2`) | ✅ Approved | SSPL permits internal deployment and operation; no redistribution or SaaS offering is involved. License terms respected as-is. |
| **HashiCorp Vault** | `BUSL-1.1` | ✅ Approved | Internal production deployment is within BUSL-1.1 terms for an integrator operating its own infrastructure. Converts to `MPL-2.0` 4 years after each release. License terms respected as-is. |

**Alternatives kept on file** in case of future license tightening:

| Flagged tool | OSS alternative | Notes |
|---|---|---|
| Redis | **Valkey** (BSD-3-Clause) or **KeyDB** (BSD-3-Clause) | Linux Foundation community forks, drop-in API compatible |
| HashiCorp Vault | **OpenBao** (MPL-2.0) | Linux Foundation fork of Vault; feature parity maintained |
| HashiCorp Terraform | **OpenTofu** (MPL-2.0) | Linux Foundation fork of Terraform; already referenced in `infra/0003-terraform-standard §Revision triggers` |

If any of the flagged licenses tighten in a future version (e.g. SSPL scope expands, BUSL converts to a stricter license before the MPL conversion date), the tool is migrated to the OSS alternative.

### 5. Bill of Materials (BoM) tracking

Every tool deployed on the platform has a mandatory `docs/licensing.md` file in its repository (or in `platform-setup/tools/{tool}/docs/licensing.md` for tools that don't have a dedicated repo). The file contains:

| Field | Required | Notes |
|---|---|---|
| `SPDX License` | ✅ | Use the official SPDX identifier — never "open source" or free text |
| `License URL` | ✅ | Direct link to the LICENSE file in the tool's upstream repo |
| `Source URL` | ✅ | Upstream project URL (GitHub, GitLab, etc.) |
| `Version constraint` | ✅ | What version or version range is approved (e.g. "Redis < 7.4" vs "Redis ≥ 7.4 SSPL") |
| `Assessment issue` | only for flagged licenses | Link to the GitHub issue where the flagged-license assessment was made |
| `Last reviewed` | ✅ | Date of last license review — reviewed annually at minimum |
| `BoM ref` | ✅ | Reference to the platform BoM entry where this tool appears |

**Platform BoM:** the full tool inventory with license columns is maintained as part of `infra/0001-platform-stack §Cross-reference`. When a tool is added or removed from the platform, both its `docs/licensing.md` and the platform BoM entry are updated in the same PR.

### 6. Review cadence

- **Per-tool:** every tool's `docs/licensing.md` is reviewed **annually** — verified that the SPDX ID is still current, the assessment (if flagged) is still valid, and no upstream license change has occurred
- **Platform-wide:** the full BoM is audited **annually** — every tool listed has an up-to-date `docs/licensing.md`, every flagged license has a current assessment
- **Event-driven:** when a vendor announces a license change (Redis → SSPL, Vault → BUSL, etc.), affected tools are re-assessed **immediately** and the outcome recorded

### 7. What this ADR does NOT cover

- **Customer-facing deployments** — if a tool is ever deployed for or resold to a customer, a separate legal review is required, outside the scope of this ADR
- **Proprietary / commercial software procurement** — this ADR covers OSS licensing only; commercial licenses have their own procurement process
- **Copyright ownership of BY-SYSTEMS code** — this ADR governs third-party tools only; BY-SYSTEMS's own code licensing policy is a separate concern
- **Export control / ECCN** — licensing is separate from export classification

## Consequences

- **Every tool has a documented SPDX license** — no silent deployments, no "we think it's open source"
- **Flagged licenses require explicit assessment** — BUSL / SSPL / AGPL / ELv2 / Commons Clause never slip in unnoticed
- **Internal-use model simplifies BUSL / SSPL / AGPL decisions** — we're an integrator, not a SaaS vendor, so redistribution and network-copyleft clauses don't trigger
- **OSS alternatives are documented** for every flagged license — if upstream tightens, the migration target is already known (Valkey for Redis, OpenBao for Vault, OpenTofu for Terraform)
- **BoM audit trail** — annual review + event-driven re-assessment + ciso-assistant integration gives ISO 27001 A.5.20 and NIS2 Art. 21(2)(d) evidence
- **Review cadence is defined** — no tool has an "expired" license assessment on file

## Revision triggers

Revise this ADR when:
- A new SPDX license category appears on the platform and doesn't fit existing approved / flagged lists
- A flagged license changes (upstream relicense, version bump with new terms, conversion date reached)
- BY-SYSTEMS's business model changes from systems integrator to SaaS vendor or software distributor — triggers re-assessment of every license on file
- A new compliance framework (DORA, SOC 2, PCI-DSS) requires different license tracking evidence
- An OSS alternative listed in §4 is itself deprecated or diverges enough that it's no longer a viable migration target

## CISO mapping

| Framework | Controls covered |
|---|---|
| ISO 27001:2022 | A.5.20 (addressing information security within supplier agreements — license policy defined for every supplier), A.8.30 (outsourced development — source available for all deployed OSS tools, license tracked per tool), A.5.23 (use of cloud services — all tools self-hosted, cloud-licensing clauses not triggered) |
| NIS2 | Art. 21(2)(d) (supply chain security — license review required per tool before deployment, BoM maintained) |
