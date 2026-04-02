<!--
| Field        | Value            |
|--------------|------------------|
| Created      | 2026-04-02       |
| Last updated | 2026-04-02       |
| Updated by   | Opus             |
-->

# Compliance Standard

**Status:** Accepted
**Date:** 2026-04-02
**Owner:** @yboujraf
**ADR:** ADR-0013

## Rule

Every infrastructure component and ADR must map its compliance posture against ISO 27001:2022, NIS2, and GDPR using the structured table format defined here. Controls are assessed as Covered, Partial, or Gap — never invented statuses.

## Requirements

1. Every infra ADR MUST include a CISO mapping section using the structured table format (see template below).
2. Only controls directly relevant to the ADR's scope are listed — do not map every ISO 27001 Annex A control.
3. Status MUST be one of: `✓ Covered` / `⚠ Partial` / `✗ Gap`.
4. `⚠ Partial` MUST include a note explaining what is missing.
5. `✗ Gap` MUST include a compensating control or `[OWNER TO DEFINE: remediation plan]`.
6. Each tool's `docs/compliance.md` contains the full per-tool compliance mapping.
7. The platform-level compliance registry is maintained in ADR-0013.
8. GDPR sections MUST only be completed for services that process personal data.

## Structured CISO table format

```markdown
## CISO mapping

> Applies only to controls directly relevant to this ADR's scope.
> Do NOT list every ISO control — only those this ADR satisfies, partially satisfies, or gaps.

### ISO/IEC 27001:2022

| Control | Title | Status | Notes |
|---|---|---|---|
| A.X.XX | Control title | ✓ Covered / ⚠ Partial / ✗ Gap | One-line explanation |

### NIS2 (Directive 2022/2555)

| Article | Requirement | Status | Notes |
|---|---|---|---|
| Art. XX(X) | Requirement text | ✓ / ⚠ / ✗ | One-line explanation |

### GDPR (Regulation 2016/679)

| Article | Requirement | Status | Notes |
|---|---|---|---|
| Art. XX | Requirement text | ✓ / ⚠ / ✗ | One-line explanation |
```

## Compliance table

| Requirement | Test | Pass condition |
|---|---|---|
| CISO section present | Check each infra ADR for `## CISO mapping` heading | Section exists |
| Correct status values | Grep for invalid statuses | Only `✓`, `⚠`, `✗` used |
| Partial has notes | Audit Partial rows | Each `⚠` row has a non-empty Notes cell |
| Gap has remediation | Audit Gap rows | Each `✗` row has compensating control or `[OWNER TO DEFINE]` |
| GDPR only where relevant | Check GDPR sections | Only present in ADRs touching personal data |

## Override procedure

To override this standard for a specific tool:
1. Create `tools/{tool}/docs/override-compliance.md`
2. State: What is different / Why / Compensating control / Reviewed by @yboujraf
3. PR must include override doc before merge is allowed

## References

- [ISO/IEC 27001:2022](https://www.iso.org/standard/27001)
- [NIS2 Directive 2022/2555](https://eur-lex.europa.eu/legal-content/EN/TXT/?uri=CELEX%3A32022L2555)
- [GDPR Regulation 2016/679](https://eur-lex.europa.eu/legal-content/EN/TXT/?uri=CELEX%3A32016R0679)
- [CISO Assistant (intuitem)](https://github.com/intuitem/ciso-assistant-community)
