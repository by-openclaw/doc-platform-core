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

## CISO mapping

> Applies only to controls directly relevant to this ADR's scope.

### ISO/IEC 27001:2022

| Control | Title | Status | Notes |
|---|---|---|---|
| A.8.6 | Capacity management | ⚠ Partial | Single PostgreSQL instance is a PoC trade-off; Patroni HA planned for Phase 2 |
| A.8.13 | Information backup | ⚠ Partial | `pg_dump` and Redis RDB snapshot strategy defined; automation not yet implemented |
| A.8.27 | Secure system architecture | ✓ Covered | Per-service databases; least-privilege users; no schema-sharing between services |

### NIS2 (Directive 2022/2555)

| Article | Requirement | Status | Notes |
|---|---|---|---|
| Art. 21(2)(c) | Business continuity | ⚠ Partial | Single PostgreSQL is a known single point of failure in PoC; Patroni HA deferred to Phase 2 |

### GDPR (Regulation 2016/679)

| Article | Requirement | Status | Notes |
|---|---|---|---|
| Art. 32(1)(b) | Integrity, confidentiality of processing | ✓ Covered | Per-service DB users with least-privilege grants; no cross-service data access |

## Consequences

- `vm-postgres-poc-01` and `vm-redis-poc-01` are critical path — all dependent services fail if either is unavailable.
- Backup strategy must cover both VMs. PostgreSQL: `pg_dump` per database + WAL archiving (Phase 2). Redis: `BGSAVE` or RDB snapshot.
- A PostgreSQL failure affects all services simultaneously — acceptable for PoC, must be mitigated in Phase 2 with Patroni.
- Simplified ops: one PostgreSQL to monitor, tune, and back up.
- Service isolation is enforced via database-level permissions, not instance separation.

## References

- `docs/stack.md` — PostgreSQL (Patroni), Redis (Sentinel) entries
- Secret storage convention defines per-service DB credential format (Phase 1 JSON)
- Vault KV path convention defines the target path format: `secret/{env}/{service}/db-password`
- `brainstorming/2026-04-02-opus-batch1-feedback.md` — authoritative runtime IP mapping
