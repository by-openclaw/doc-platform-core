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

## Scoped ADRs (refactor in progress — see [REFACTOR-DRAFT.md](REFACTOR-DRAFT.md))

| Scope | Folder | Status |
|---|---|---|
| Identity (authentication, provisioning, machine creds, OS accounts) | [`identity/`](identity/) | Draft — 4 ADRs |
| Git (workflow, platform strategy, configuration) | [`git/`](git/) | Draft — 3 ADRs |
| Naming (infra, identity, firewall, automation) | [`naming/`](naming/) | Draft — 4 ADRs |
| Security (secret storage, compliance, hardening, certificates) | [`security/`](security/) | Draft — 4 ADRs |

Flat ADRs are being migrated to scoped folders one scope at a time. See the refactor draft for the full plan.

## Records (flat — legacy)

| ADR | Title | Status | Date |
|---|---|---|---|
| [ADR-0001](0001-platform-stack-decisions.md) | Platform stack decisions | Accepted | 2026-03-29 |
| [ADR-0002](0002-repository-structure-and-naming.md) | Repository structure and naming | Accepted | 2026-03-29 |
| [ADR-0003](0003-issue-tracking-standard.md) | Issue tracking standard | Accepted | 2026-03-29 |
| [ADR-0006](0006-platform-charter.md) | Platform charter (layer model) | Accepted | 2026-03-30 |
| [ADR-0007](0007-automation-scripting-standard.md) | Automation & scripting standard | Accepted | 2026-03-30 |
| [ADR-0008](0008-terraform-state-management.md) | Terraform state management | Accepted | 2026-03-30 |
| [ADR-0009](0009-netbox-cmdb.md) | NetBox as CMDB source of truth and intent layer | Accepted | 2026-03-31 |
| [ADR-0020](0020-backup-strategy.md) | Platform backup strategy | Draft | 2026-04-02 |
| [ADR-0021](0021-hardening-standard.md) | Platform hardening baseline | Draft | 2026-04-02 |
| [ADR-0022](0022-licensing-policy.md) | Open source licensing policy | Draft | 2026-04-02 |
| [ADR-0023](0023-monitoring-approach.md) | Monitoring approach (Prometheus + Grafana) | Accepted | 2026-04-02 |
| [ADR-0012](0012-environment-tier-standard.md) | Environment tier standard | Accepted | 2026-03-31 |
| [ADR-0015](0015-network-vlan-architecture.md) | Network VLAN architecture | Draft | 2026-04-02 |
| [ADR-0017](0017-logging-standard.md) | Logging standard (Docker JSON, Promtail→Loki) | Draft | 2026-04-02 |
| [ADR-0018](0018-centralized-database-strategy.md) | Centralized database strategy (PostgreSQL + Redis) | Draft | 2026-04-02 |
