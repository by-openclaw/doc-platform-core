# services/0004 — Database Strategy

**Status:** Draft
**Date:** 2026-04-14 (supersedes flat ADR-0018, 2026-04-02)
**Scope:** Centralized PostgreSQL and Redis for platform services — who consumes them, how they are isolated, how credentials flow, and the HA roadmap. Does not define backup mechanics (see `infra/0008-backup-strategy`), exporter configuration (see `infra/0007-monitoring`), or specific VM names / IPs (those live in NetBox).
**Related:** `security/0001-secret-storage`, `services/0003-netbox-cmdb`, `infra/0007-monitoring`, `infra/0008-backup-strategy`, `services/0002-email-infrastructure`

---

## Context

Multiple platform services (Authentik, GitLab CE, NetBox, Vaultwarden, Nextcloud) require PostgreSQL as their primary relational database. Two of them (Authentik, GitLab CE) also require Redis for sessions and caching. Running a dedicated database VM **per service** would fragment state management, multiply backup and monitoring surface, and waste resources at the platform scale.

At the same time, **sharing a PostgreSQL instance between unrelated services is a least-privilege violation** unless the isolation is enforced at the database layer itself. And running a **single instance** of either PostgreSQL or Redis creates a platform-wide single point of failure — every shared-DB consumer fails simultaneously if one VM goes down. This is unacceptable for the platform services that need to survive maintenance windows and hardware failures.

The policy must be explicit about both concerns: **shared but isolated** at the data layer, **clustered** at the infrastructure layer.

## Decision

### Clustered PostgreSQL + clustered Redis, from day 1, with per-service isolation

- **PostgreSQL cluster** (Patroni, 3 nodes) hosts **all platform services that use PostgreSQL**. No single-instance deployment.
- **Redis cluster** (Sentinel, 3 sentinels + primary + replica) hosts **all platform services that use Redis**. No single-instance deployment.
- Each service gets its own **database** (not schema) on PostgreSQL. No schema-sharing between services.
- Each service gets its own **database user** with grants scoped to **its own database only**. Cross-service queries are impossible at the database layer.
- Each service that uses Redis gets its own **logical database number** (`SELECT n` per client connection) or its own key prefix, per service's native support.

**Rule:** no platform service depends on a single-instance PostgreSQL or Redis. A node loss must not cause service outage — it must cause at most a brief reconnection pause while the cluster re-elects.

### Cluster placement — single Proxmox host

**All cluster VMs (Patroni nodes, pooler, Redis nodes, Sentinels, Patroni DCS) run on the same Proxmox hypervisor.** This reflects the current platform state (single Proxmox host at the `br` site per `naming/0001-infra §4` site codes). Anti-affinity across physical hosts is **not available** until a second Proxmox host or a cluster is deployed.

**What the cluster DOES protect against (with all VMs on one host):**

- **Process crashes** — PostgreSQL / Redis / Patroni / Sentinel process dies, the cluster elects a new leader within seconds, clients reconnect
- **OS reboots** — a VM is rebooted for kernel patching or Ansible role upgrades, the cluster reshapes around it, no service outage
- **Rolling upgrades** — PostgreSQL minor version bump, Redis upgrade, `os-qemu-guest-agent` update, all done node-by-node with zero client-visible downtime
- **Failover drills** — Patroni `patronictl failover` and Redis Sentinel `SENTINEL FAILOVER` can be exercised regularly without scheduling a maintenance window
- **Misconfiguration / split-brain recovery** — the DCS (etcd / Consul) and Sentinel quorum protect against inconsistent state even in the presence of operator error

**What the cluster does NOT protect against (single-host limitation):**

- **Proxmox host failure** — if the physical host dies, every VM on it dies simultaneously. The cluster cannot self-heal because there are no surviving nodes.
- **Storage failure** — shared Proxmox storage (ZFS, LVM) failure takes down every VM sharing it.
- **Site failure** — network outage, power loss, physical damage at the `br` site.

**How the single-host risk is mitigated (without adding a second host):**

| Risk | Mitigation |
|---|---|
| Hardware failure | `drp` env tier per `infra/0005-environment-tiers` — separate Patroni + Sentinel clusters on different infrastructure (or rented VPS), populated via backup restore per `infra/0008-backup-strategy` |
| Storage failure | ZFS snapshots + Proxmox Backup Server to Synology NAS (per `infra/0008-backup-strategy`) — recovery path is restore, not continuous failover |
| Rolling upgrades to Proxmox itself | Schedule a maintenance window (no avoiding this with one host) |

**Revision trigger (important):** when a second Proxmox host is added, the cluster VMs must be **distributed across hosts** via Proxmox HA groups / anti-affinity rules. At that point this ADR is amended to reflect true multi-host HA.

Until then, **HA in this ADR means "software-level HA"**: it eliminates software-failure-induced outages, enables zero-downtime upgrades, and provides operationally useful failover — but the single physical host is a known and accepted limitation, tracked in `security/0002-compliance-mapping §Known gaps`.

### Cluster architecture

#### PostgreSQL — Patroni cluster (3 nodes)

| Node | Role | Purpose |
|---|---|---|
| `vm-pgsql-01` | Leader (primary) | Read + write |
| `vm-pgsql-02` | Sync replica | Synchronous streaming replication — promoted on leader failure |
| `vm-pgsql-03` | Async replica | Asynchronous replication — read-only replica, standby for maintenance |

**Frontend:** a separate pooler VM (`vm-pgbouncer-01` or `vm-pgpool-01`, choice deferred per §Deferred decisions) fronts the cluster with a **single connection endpoint**. Services connect to the pooler hostname, not to individual Patroni nodes. On leader failover, Patroni updates the pooler's routing automatically via its REST API.

**Consensus:** Patroni uses a DCS (distributed configuration store) for leader election. Options: **etcd** (3-node cluster) or **Consul** (if already deployed). Consensus tier is deferred per §Deferred decisions.

**Replication:** `synchronous_standby_names = 'ANY 1 (vm-pgsql-02, vm-pgsql-03)'` — at least one replica must confirm the write before the primary acknowledges. No lost transactions on leader failure.

#### Redis — Sentinel cluster (3 sentinels + primary + replica)

| Node | Role |
|---|---|
| `vm-redis-01` | Primary |
| `vm-redis-02` | Replica (promoted on primary failure) |
| `vm-redis-sentinel-01` / `-02` / `-03` | Sentinel quorum (3 nodes for majority) |

**Sentinel deployment:** the 3 Sentinel processes may be **colocated** with the 2 Redis nodes + 1 standalone Sentinel VM, to keep VM count at 3 instead of 5. The colocation is fine for Sentinel — its role is monitoring and leader election, not data handling.

**Client-side discovery:** services use a Sentinel-aware Redis client (Python `redis-py`, Ruby `redis`, Go `go-redis` — all support Sentinel natively). The client is configured with the Sentinel list, not with the Redis primary directly. On primary failure, the Sentinel quorum elects the replica as new primary and clients reconnect automatically.

**Quorum:** 2 of 3 Sentinels must agree to trigger failover. This tolerates 1 Sentinel loss without split-brain.

### Naming

Per `naming/0001-infra §5`, all cluster nodes follow `{site}-{service}-{seq:02d}`:

| Component | Canonical name example | NetBox role |
|---|---|---|
| PostgreSQL Patroni node 1 (leader) | `vm-pgsql-01` | `db` |
| PostgreSQL Patroni node 2 (sync replica) | `vm-pgsql-02` | `db` |
| PostgreSQL Patroni node 3 (async replica) | `vm-pgsql-03` | `db` |
| PostgreSQL frontend pooler | `vm-pgpool-01` | `db` |
| Redis primary | `vm-redis-01` | `cache` |
| Redis replica | `vm-redis-02` | `cache` |
| Redis Sentinel (3 nodes, may colocate) | `vm-sentinel-01` / `-02` / `-03` | `cache` |
| etcd consensus tier (3 nodes, if Patroni uses etcd) | `vm-etcd-01` / `-02` / `-03` | `db` (or `mon` — deferred) |

The actual names are whatever each deployment calls them per NetBox. This ADR does not hardcode specific names or IPs — both come from `services/0003-netbox-cmdb`.

The role at leader-vs-replica level is tracked in NetBox as a **tag** (`patroni-leader`, `patroni-sync`, `patroni-async`, `redis-primary`, `redis-replica`) or as a dynamic status from Patroni's / Sentinel's API — not as a hostname suffix (hostnames are static, cluster roles change on failover).

### Per-service database table

Every service that consumes the shared PostgreSQL has:

1. Its own database (name = service short code from `naming/0001-infra §5`)
2. Its own dedicated user (`{service}_app` or similar — per-service convention allowed, tracked in NetBox)
3. Grants: `CONNECT` + `USAGE` on its own schema, `SELECT/INSERT/UPDATE/DELETE` on its own tables — **nothing else**
4. Credentials stored in HashiCorp Vault at `secret/{env}/{service}/db-password` per `security/0001-secret-storage`

| Service | Database name | Purpose |
|---|---|---|
| Authentik | `authentik` | User accounts, group memberships, OIDC/SAML providers, tokens |
| GitLab CE | `gitlab` | Projects, issues, MRs, CI jobs, users, groups |
| NetBox | `netbox` | CMDB records — the authoritative platform catalog |
| Vaultwarden | `vaultwarden` | User vault metadata (encrypted vault data is stored in Vaultwarden's own structure, not raw in PG) |
| Nextcloud | `nextcloud` | File metadata, shares, user accounts (when Nextcloud is deployed) |

**Adding a new PostgreSQL consumer** = add a row to this table via ADR amendment.

### Redis consumers

| Service | Redis DB / prefix | Purpose |
|---|---|---|
| Authentik | `select 0` | Session cache, background task queue |
| GitLab CE | `select 1` | Session cache, Sidekiq job queue |

**Redis is cache / session / queue only.** No durable data is stored in Redis — if Redis loses state, services rebuild from PostgreSQL. Services that need durable key-value storage use PostgreSQL, not Redis.

### Non-consumers — services that do NOT use the shared instances

These services are listed explicitly to prevent accidental "let's connect them to shared PG too" decisions:

| Service | Database / cache | Why not shared |
|---|---|---|
| **Mailcow** | Internal MariaDB + Redis (official image) | Mailcow ships with its own MariaDB + Redis in the `mailcow/mailcow-dockerized` compose file. Reconfiguring it to use external DB/Redis is off the supported upgrade path — see `services/0002-email-infrastructure` §Mailcow deployment. |
| **OPNsense** | None | Firewall state is in OPNsense's own FreeBSD-native storage. No external DB. |
| **Prometheus** | Own TSDB on local disk | Time-series data model is not relational; PostgreSQL is unsuitable. |
| **Loki** | Object store (MinIO / Contabo S3) | Log data uses an object-store schema, not relational. See `infra/0006-logging`. |
| **Traefik** | None | Config is static / API, not database-backed. |
| **Vault (HashiCorp)** | Own Raft storage | Vault's integrated Raft is the supported backend. Using PostgreSQL as Vault backend is supported by HashiCorp but is not used on this platform. |
| **step-ca** | File-based | step-ca's BadgerDB / file backend is sufficient. |
| **Grafana** | SQLite default (small scale) or shared PostgreSQL (if scale requires) | **Decision deferred** — start with SQLite (simplest, native), migrate to shared PostgreSQL if Grafana grows beyond single-instance. If migrated, Grafana is added to the §Per-service database table and to `services/0003-netbox-cmdb §3 role=mon` entry. |

**Rule:** a service is a consumer only if it appears in the §Per-service database table (PostgreSQL) or the §Redis consumers table. Everything else is explicitly a non-consumer.

### Credential flow

Credentials never appear in git, in docker-compose files, in environment variables, or in service config outside of Vault:

```
1. Ansible creates the database + user during Layer 5 service deployment
   (role: shared-db-provisioning — reads service list from NetBox)

2. Ansible generates a random password (≥32 chars, alphanumeric + symbols)

3. Ansible writes the password to Vault at:
     secret/{env}/{service}/db-password

4. Ansible configures the service (docker-compose, systemd, Helm) to read
   the credential at runtime via Vault Agent sidecar or environment
   injection — never from a static file on disk.

5. Password rotation is a one-command Ansible task:
     - generate new password
     - ALTER USER on PostgreSQL
     - update Vault
     - restart the service (service re-reads Vault at startup)
```

All grants are applied with `GRANT ... ON DATABASE <service>` and `GRANT ... ON ALL TABLES IN SCHEMA public TO {service}_app`. No `GRANT ALL PRIVILEGES ON ALL DATABASES` anywhere.

### Failover behavior

**PostgreSQL (Patroni):**
- Leader failure detected by Patroni in seconds via DCS lease expiration
- Sync replica is promoted to leader (no data loss because synchronous replication already confirmed the write)
- Pooler front-end updates routing via Patroni REST API
- Client impact: a brief reconnection pause (seconds) — no application change, no manual intervention
- Failed node is marked unhealthy in NetBox via Ansible reconciliation

**Redis (Sentinel):**
- Primary failure detected by Sentinel quorum
- Replica promoted to primary by Sentinel majority vote
- Sentinel-aware clients receive the new primary address via subscription to `__sentinel__:hello` and reconnect automatically
- Client impact: a brief reconnection pause — the brief loss of in-flight cache data is acceptable because Redis is cache, not source of truth

### Disaster recovery (`drp` env)

The `drp` environment (per `infra/0005-environment-tiers`) runs its own Patroni cluster and its own Redis Sentinel cluster — **not** as replicas of the prod clusters. DR activation is a full restore from backup (per `infra/0008-backup-strategy`), not a continuous replication tail.

**Rationale:** continuous cross-env replication would break the env-isolation invariant — a prod leak would automatically flow to DR. DR is populated from encrypted backups, not from live prod traffic.

### Backup

Backup is owned by `infra/0008-backup-strategy`. The database strategy ADR does not define backup policy — it just confirms:

- PostgreSQL falls into the **"Relational DB"** backup class (`pg_dump` + WAL archiving)
- Redis falls into the **"Cache / queue"** backup class (RDB snapshots, best effort — Redis is cache, not source of truth, per §Redis consumers)
- Per-service RTO / retention / schedule are the §Pending decisions in `infra/0008-backup-strategy`

### Monitoring

Per `infra/0007-monitoring §Scrape requirement`:

| Service | Exporter |
|---|---|
| PostgreSQL | `postgres_exporter` on the DB VM |
| Redis | `redis_exporter` on the Redis VM |

Alert baseline (minimum — per `infra/0007-monitoring §Alert baseline`):

- Service down (target unreachable)
- Connection pool saturation (PostgreSQL `pg_stat_activity` near `max_connections`)
- Replication lag (Phase 2 HA only)
- Cache hit ratio (Redis — warning if < 85%)
- Disk space (WAL / RDB — warning at 80%, critical at 90%)

### Break-glass

- Each database has a break-glass superuser account stored in Vault at `secret/{env}/pgsql/break-glass` and `secret/{env}/redis/break-glass`
- Break-glass access is audited — any `vault kv read` of these paths is logged and alerts on Discord critical channel per `infra/0007-monitoring §Alert routing`
- The break-glass account is only ever used during incident response and is rotated immediately after use per `security/0001-secret-storage §Rotation`

## Consequences

- **Software HA is the baseline, not a future upgrade.** PostgreSQL runs as a 3-node Patroni cluster from day 1. Redis runs as Sentinel (3 sentinels + primary + replica). No single-instance deployment exists at any env tier above `dev`.
- **All cluster VMs run on a single Proxmox host** today — this eliminates software-failure outages, enables zero-downtime rolling upgrades, and enables failover drills, **but does not protect against hardware failure of the host itself**. Hardware-failure mitigation is via `drp` env + backup restore, not via the cluster.
- **Anti-affinity becomes real when a second Proxmox host is added** — this ADR is amended at that point to require the cluster VMs to be distributed across hosts.
- **Resource footprint is higher** — at least **3 PostgreSQL VMs + 1 pooler + 2 Redis VMs + 3 Sentinel instances (colocatable) + 3 DCS nodes (colocatable)** = 6 to 12 VMs on a single Proxmox host for the DB/cache layer alone. The trade-off is made consciously: the platform supports Authentik, GitLab, NetBox — a software-level DB outage means every engineer is locked out of every tool simultaneously. Software HA is cheaper than the incident that would otherwise happen.
- **Every PG-consuming service** has its own database + user + credentials in Vault. Cross-service queries are blocked at the database layer.
- **Mailcow is excluded by design** — it uses its internal MariaDB/Redis from the official image. Reconnecting Mailcow to shared DB breaks the supported upgrade path.
- **Failover is automatic.** Patroni promotes sync replica on leader loss, Sentinel promotes Redis replica on primary loss, pooler / clients reconnect. No manual intervention except to replace the failed node later.
- **No lost writes on PG leader failure** — synchronous replication to at least one standby is enforced.
- **DR (`drp`) is a separate cluster**, populated by backup restore rather than live replication — preserves env isolation per `infra/0005-environment-tiers`.
- **Monitoring, backup, and role vocabulary** are owned by other ADRs. This one only references them.
- **Adding a new consumer requires an ADR amendment** (update the §Per-service database table) — prevents accidental "we'll just connect this new tool to shared PG" decisions.
- **Dev env exception:** a `dev` env tier deployment may choose to run single-instance PostgreSQL and Redis to reduce resource cost — this is the **only** exception. All `test`, `staging`, `acc`, `prod`, and `drp` deployments run the full cluster per this ADR.

## Deferred decisions

These are intentionally empirical — resolve at deployment time, not in this ADR.

| Decision | Options |
|---|---|
| PostgreSQL frontend pooler | `pgbouncer` (lightweight, transaction pooling) **or** `pgpool-II` (richer routing, heavier) |
| Patroni DCS | `etcd` (3-node cluster, dedicated) **or** `Consul` (if Consul is already deployed for service discovery) |
| Sentinel colocation | 3 dedicated Sentinel VMs **or** 1 dedicated + 2 colocated on Redis nodes |
| Redis replica count | Minimum 1 replica (2 total Redis nodes) vs 2 replicas (3 total Redis nodes) — deferred until load testing |

## Revision triggers

Revise this ADR when:
- **A second Proxmox host is added** — the cluster gains true hardware anti-affinity. Update §Cluster placement to require VM distribution across hosts via Proxmox HA groups.
- A new service needs PostgreSQL or Redis (add to §Per-service database table or §Redis consumers)
- A service is explicitly moved **off** the shared instances (rare — would be a major deployment change)
- PostgreSQL is replaced by a different relational DB (very unlikely)
- Redis is replaced by a different cache / queue (more likely — e.g. Valkey fork, KeyDB)
- Grafana is migrated from SQLite to shared PostgreSQL (add Grafana to §Per-service database table)
- Vaultwarden encrypted vault storage moves to PostgreSQL (currently uses its own structure)
- Patroni is replaced by a different HA mechanism (e.g. CloudNativePG if the platform moves fully to Kubernetes)
- Any §Deferred decision is resolved

## CISO mapping

| Framework | Controls covered |
|---|---|
| ISO 27001:2022 | A.8.6 (capacity management — ⚠ partial, single instance is a PoC trade-off), A.8.13 (backup — delegated to `infra/0008`), A.8.27 (secure system architecture — per-service DB users with least-privilege grants, no schema sharing) |
| NIS2 | Art. 21(2)(c) (business continuity — ⚠ partial, single PostgreSQL is a known SPOF until Phase 2 Patroni HA) |
| GDPR | Art. 32(1)(b) (integrity, confidentiality — per-service DB users with least-privilege grants prevent cross-service data access) |
