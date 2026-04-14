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
| Infra (stack, charter, terraform, network, env tiers, logging, monitoring, backup) | [`infra/`](infra/) | Draft — 8 ADRs |
| Services (opnsense, email, netbox CMDB, database strategy) | [`services/`](services/) | Draft — 4 ADRs |

Flat ADRs are being migrated to scoped folders one scope at a time. See the refactor draft for the full plan.

## Records (flat — legacy, awaiting scope refactor)

| ADR | Title | Planned scope |
|---|---|---|
| [ADR-0002](0002-repository-structure-and-naming.md) | Repository structure and naming | Split: naming part already in `naming/0004-automation`; structure part → `OPERATING-STANDARD.md` |
| [ADR-0003](0003-issue-tracking-standard.md) | Issue tracking standard | Archive — already in `OPERATING-STANDARD.md §7` |
| [ADR-0007](0007-automation-scripting-standard.md) | Automation & scripting standard | Archive — already in `OPERATING-STANDARD.md` |
| [ADR-0022](0022-licensing-policy.md) | Open source licensing policy | `security/0005-licensing-policy` (follow-up — licensing is a compliance concern, not a service) |
