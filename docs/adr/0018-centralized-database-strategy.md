# ADR-0018: Centralized Database Strategy

- **Status:** Draft
- **Date:** 2026-04-02
- **Deciders:** @yboujraf

## Context

Multiple platform services (Authentik, GitLab, NetBox, Grafana, Nextcloud, Odoo) require PostgreSQL and/or Redis. A decision is needed on whether each service runs its own database instance or whether a shared centralized PostgreSQL cluster (with per-service schemas/credentials) and Redis instance are used — with a defined HA roadmap (Patroni, Redis Sentinel).

## Decision

TODO: pending @yboujraf review

## Consequences

TODO: pending decision

## References

- `docs/stack.md` — PostgreSQL (Patroni), Redis (Sentinel) entries
- ADR-0011 — Secret storage for per-service DB credentials (`secret/svc/postgresql/*`)
- `docs/archive/2026-04-01-vault-kv-standard.md` — per-service credential paths
- `brainstorming/2026-04-02-doc-matrix.md` — missing ADR identified here
</content>