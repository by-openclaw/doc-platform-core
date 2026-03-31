# Architecture Decision Records

Platform-wide architectural decisions for BY-SYSTEMS.

Repo-scoped decisions live in each repo's `docs/adr/` directory (e.g., `lib-synology-dsm/docs/adr/`).

> When referencing across scopes, use the full path `doc-platform-core/docs/adr/NNNN-...` to disambiguate from repo-scoped ADRs.

## Format

```
NNNN-short-title.md
```

```markdown
# ADR-NNNN: Title
**Date:** YYYY-MM-DD
**Status:** Proposed | Accepted | Deprecated | Superseded by ADR-XXXX

## Context
## Decision
## Consequences
## Compliance
```

## Records

| ADR | Title | Status | Date |
|---|---|---|---|
| [ADR-0001](0001-platform-stack-decisions.md) | Platform stack decisions | Accepted | 2026-03-29 |
| [ADR-0002](0002-repository-structure-and-naming.md) | Repository structure and naming | Accepted | 2026-03-29 |
| [ADR-0003](0003-issue-tracking-standard.md) | Issue tracking standard | Accepted | 2026-03-29 |
| [ADR-0004](0004-identity-sso-architecture.md) | Identity & SSO architecture | Accepted | 2026-03-29 |
| [ADR-0005](0005-vcs-and-cicd-strategy.md) | VCS and CI/CD strategy | Accepted | 2026-03-29 |
| [ADR-0006](0006-platform-charter.md) | Platform charter (layer model) | Accepted | 2026-03-30 |
| [ADR-0007](0007-automation-scripting-standard.md) | Automation & scripting standard | Accepted | 2026-03-30 |
| [ADR-0008](0008-terraform-state-management.md) | Terraform state management | Accepted | 2026-03-30 |
| [ADR-0009](0009-netbox-as-cmdb-source-of-truth-and-intent-layer.md) | NetBox as CMDB source of truth and intent layer | Accepted | 2026-03-31 |
| [ADR-0010](0010-naming-and-identity-convention.md) | Naming & identity convention | Accepted | 2026-03-31 |
| [ADR-0011](0011-secret-storage-convention.md) | Secret storage convention | Accepted | 2026-03-31 |
| [ADR-0012](0012-environment-tier-standard.md) | Environment tier standard | Accepted | 2026-03-31 |
| [ADR-0013](0013-compliance-framework-mapping.md) | Compliance framework mapping (first pass) | Accepted | 2026-03-31 |
