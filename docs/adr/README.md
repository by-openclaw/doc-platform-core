# Architecture Decision Records

Platform-wide architectural decisions for BY-SYSTEMS.

Repo-scoped decisions live in each repo's `docs/adr/` directory (e.g., `lib-synology-dsm/docs/adr/`).

> When referencing across scopes, use the full path `doc-platform-core/docs/adr/{scope}/NNNN-...` to disambiguate from repo-scoped ADRs.

## Format

```
{scope}/NNNN-short-title.md
```

```markdown
# {scope}/NNNN — Title
**Status:** Draft | Accepted | Deprecated | Superseded by {scope}/NNNN
**Date:** YYYY-MM-DD
**Scope:** one-line description of what this ADR owns
**Related:** links to peer ADRs in other scopes

## Context
## Decision
## Consequences
## CISO mapping
```

## Scoped ADRs

| Scope | Folder | ADRs |
|---|---|---|
| Identity (authentication, provisioning, machine creds, OS accounts) | [`identity/`](identity/) | 4 |
| Git (workflow, platform strategy, configuration) | [`git/`](git/) | 3 |
| Naming (infra, identity, firewall, automation) | [`naming/`](naming/) | 4 |
| Security (secret storage, compliance, hardening, certificates, licensing) | [`security/`](security/) | 5 |
| Infra (stack, charter, terraform, network, env tiers, logging, monitoring, backup) | [`infra/`](infra/) | 8 |
| Services (opnsense, email, netbox CMDB, database strategy, notifications) | [`services/`](services/) | 5 |
| Lib — Python (design standard) | [`lib/python/`](lib/python/) | 1 |
| **Total** | — | **30** |

## Refactor status

The scoped ADR refactor is **complete**. Every decision that was previously in a flat `NNNN-*.md` ADR now lives in one of the scope folders above, in `OPERATING-STANDARD.md`, or is preserved under [`archive/`](archive/) as a pre-refactor snapshot.

The legacy flat ADR table has been removed from this README. Historical flat ADRs are preserved as `archive/NNNN-*.pre-refactor.md` snapshots — they are read-only records of the pre-refactor state, not authoritative sources of current decisions.

## Archive

- [`archive/`](archive/) — pre-refactor snapshots of every flat ADR, captured in PR #13 before any migration began. Used for:
  - Historical context when a scoped ADR's `supersedes` line points at a flat number
  - Verification during scoped-ADR reviews that no decision was lost in translation
  - Auditor-friendly "what did we decide before the refactor" reference

Archive files are **never edited**. If a decision needs updating, update the scoped ADR that supersedes it — never the archived pre-refactor snapshot.
