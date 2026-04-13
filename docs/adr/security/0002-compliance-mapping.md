# security/0002 — Compliance Framework Mapping

**Status:** Draft
**Date:** 2026-04-13 (supersedes flat ADR-0013, 2026-03-31)
**Scope:** Governance decision — how the platform tracks compliance against ISO 27001, NIS2, GDPR, and DORA. Does not enumerate controls inline.
**Related:** `security/0001-secret-storage`, `security/0003-hardening`, `security/0004-certificate-strategy`, every ADR's `CISO mapping` section

---

## Context

BY-SYSTEMS targets four compliance frameworks — ISO 27001:2022, NIS2, GDPR, and DORA. Controls are referenced in individual ADRs (each has a `CISO mapping` section), but there is no single registry showing current coverage, partial implementations, and gaps. Without a registry, compliance posture is invisible to auditors and the team.

Maintaining inline control tables in a markdown ADR duplicates data that belongs in a purpose-built GRC tool. Tables in markdown cannot be queried, reported, or exported as evidence; they drift silently when individual ADRs change their `CISO mapping` sections.

## Decision

### ciso-assistant-community is the compliance registry

The platform uses **[intuitem/ciso-assistant-community](https://github.com/intuitem/ciso-assistant-community)** (open source, self-hosted) as the authoritative compliance registry. It holds:

- Framework catalogs (ISO 27001:2022, NIS2, GDPR, DORA) imported from their built-in libraries
- Requirement nodes per framework (tree of control objectives)
- Applied controls — the platform's implementations of those requirements
- Evidence links pointing back to individual ADRs in this repo
- Status dashboard per framework (covered / partial / gap)

**Git is the source of truth for architectural decisions (ADRs). ciso-assistant is the source of truth for compliance posture.** Neither replaces the other.

### How inputs flow from ADRs to the registry

Every ADR in this repo includes a `CISO mapping` section listing the control IDs it covers:

```markdown
## CISO mapping

| Framework | Controls covered |
|---|---|
| ISO 27001:2022 | A.5.15, A.5.16, A.5.17 |
| NIS2 | Art. 21(2)(a), 21(2)(i) |
| GDPR | Art. 32 |
```

These mappings are the **inputs** to ciso-assistant. A future importer script (out of scope for this ADR) parses the mapping sections from every ADR and creates/updates applied-control records in ciso-assistant, with each applied control linked to the ADR file as evidence.

Until the importer exists, ciso-assistant is populated manually during quarterly compliance reviews. The inline tables in each ADR's `CISO mapping` section are the canonical input form — short, auditable, and mechanically parseable.

### What this ADR does NOT do

This ADR deliberately does not:

1. **Duplicate the framework catalogs.** ISO Annex A, NIS2 articles, GDPR articles, DORA articles are imported into ciso-assistant from its built-in libraries. This markdown file would drift the moment a framework is updated.
2. **Maintain an inline control-to-ADR mapping table.** Such a table was present in flat ADR-0013 and became stale within weeks. The same data lives in each ADR's `CISO mapping` section and, once the importer exists, in ciso-assistant.
3. **Define the importer.** The importer is an implementation concern tracked as a separate follow-up issue.

### ADR authoring rule

Every new ADR **must** include a `CISO mapping` section. ADRs that do not cover any compliance control include the section with `N/A — no direct control mapping` rather than omitting it. This keeps the importer's input shape uniform.

The CISO mapping section is the only place in an ADR that lists framework control identifiers. Control IDs do not appear elsewhere in ADR prose.

## Known gaps — compliance posture at time of writing

These gaps are maintained as a **short list in this ADR** (not in ciso-assistant yet) because they track platform maturity, not individual controls. Move to ciso-assistant when it is deployed.

| Gap | Framework | Priority | Notes |
|---|---|---|---|
| No formal policy review schedule | ISO A.5.1 | MEDIUM | Annual review cadence to be defined |
| No asset inventory tool live | ISO A.5.9 / A.8.9 | HIGH | NetBox deployment pending (`services/0003-netbox-cmdb`) |
| No encryption at rest for transitional JSON secrets | ISO A.8.24 | HIGH | HashiCorp Vault deployment pending (`security/0001-secret-storage`) |
| No centralized event logging | ISO A.8.15 | HIGH | Loki + Wazuh pending (`infra/0006-logging`, future) |
| Network segmentation not yet enforced | ISO A.8.20 / A.8.22 | HIGH | OPNsense deployment pending (`services/0001-opnsense-provisioning`) |
| No formal incident response process | NIS2 Art. 21(2)(b) | HIGH | IRP document to be written |
| No BCP / DRP document | NIS2 Art. 21(2)(c) | MEDIUM | Create after core infra stable |
| No vendor risk assessment process | NIS2 Art. 21(2)(d) | LOW | Relevant when external vendors onboarded |
| No DORA mapping in ciso-assistant | DORA | LOW | Overlap with ISO / NIS2 — map after those are complete |
| No formal audit schedule | ISO A.5.35 | MEDIUM | Define after ciso-assistant deployed |
| No ciso-assistant importer | — | MEDIUM | Script to parse ADR `CISO mapping` sections and POST to ciso-assistant API |

**DORA status:** DORA is listed as a compliance target but has no inline controls because it overlaps significantly with ISO 27001 and NIS2. A dedicated DORA pass will happen once the ISO and NIS2 mappings in ciso-assistant are stable, and will map existing controls to DORA articles to identify net-new gaps only.

## Consequences

- **Compliance posture is visible in ciso-assistant**, not in markdown. Auditor-friendly, queryable, reportable.
- **ADR `CISO mapping` sections are the canonical input** — short, per-ADR, no duplication across files.
- **No inline control tables in this ADR** — nothing to drift when a framework is updated or an ADR changes its mapping.
- **Gap list is short and actionable** — 11 items, not 80. Maintained in this ADR until ciso-assistant is deployed, then migrated.
- **The importer is a separate deliverable** — this ADR does not block on it.
- **Every new ADR must include a CISO mapping section** (or `N/A` placeholder) — enforced by code review until a lint rule exists.

## Revision triggers

Revise this ADR when:
- ciso-assistant is replaced by a different GRC tool
- A fifth compliance framework is added (e.g. SOC 2, PCI-DSS) and requires a different mapping strategy
- The importer is built and the gap list migrates to ciso-assistant — this ADR may then shrink to a single "see ciso-assistant" pointer
- Framework-catalog divergence from ciso-assistant's built-in libraries forces the platform to maintain its own fork

## CISO mapping

This ADR is the governance decision for compliance mapping itself. It is meta: the only control it directly covers is the existence of a compliance registry.

| Framework | Controls covered |
|---|---|
| ISO 27001:2022 | A.5.1 (policies for information security — this ADR governs the compliance policy framework) |
| NIS2 | Art. 21(2)(a) (risk management — a compliance registry is a prerequisite for risk tracking) |
