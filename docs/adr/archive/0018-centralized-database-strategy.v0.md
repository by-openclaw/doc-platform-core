# ADR-0018: Centralized Database Strategy

- **Status:** Accepted
- **Date:** 2026-04-02
- **Deciders:** @yboujraf

## Context

Multiple platform services (Authentik, GitLab, NetBox, Vaultwarden, Nextcloud) require PostgreSQL. Running a dedicated database VM per service would fragment state management, increase resource consumption, and complicate backup. A centralized approach is evaluated against the operational overhead trade-off for PoC scale.

Redis is similarly required by multiple services (Authentik, GitLab) for caching and session storage.

## Decision

**Single PostgreSQL instance:** `vm-postgres-poc-01`, `10.1.3.19` (SVC zone).

All platform services share this instance. Each service gets its own **database** (not schema). No schema-sharing between services.

| Service | Database name |
|---|---|
| Authentik | `authentik` |
| GitLab | `gitlab` |
| NetBox | `netbox` |
| Vaultwarden | `vaultwarden` |
| Nextcloud | `nextcloud` |

Per-service database users with least-privilege grants. Credentials stored in Vault under `secret/poc/{service}/db-password`.

**Single Redis instance:** `vm-redis-poc-01`, `10.1.3.18` (SVC zone). Used for cache and session storage only. No durable data stored in Redis.

**No per-service database VMs.** Each service does not get its own PostgreSQL or Redis instance in PoC.

**HA roadmap (Phase 2):** Patroni for PostgreSQL HA, Redis Sentinel for Redis HA. Not in scope for PoC.

## Consequences

- `vm-postgres-poc-01` and `vm-redis-poc-01` are critical path — all dependent services fail if either is unavailable.
- Backup strategy must cover both VMs. PostgreSQL: `pg_dump` per database + WAL archiving (Phase 2). Redis: `BGSAVE` or RDB snapshot.
- A PostgreSQL failure affects all services simultaneously — acceptable for PoC, must be mitigated in Phase 2 with Patroni.
- Simplified ops: one PostgreSQL to monitor, tune, and back up.
- Service isolation is enforced via database-level permissions, not instance separation.

## References

- `docs/stack.md` — PostgreSQL (Patroni), Redis (Sentinel) entries
- ADR-0011 — Secret storage for per-service DB credentials
- ADR-0016 — Vault KV path format for DB passwords
- `brainstorming/2026-04-02-opus-batch1-feedback.md` — authoritative runtime IP mapping
