# infra/0008 — Backup Strategy

**Status:** Draft — several policy thresholds pending (`⚠ TBD` in §Pending decisions)
**Amended:** 2026-10-20 (approved by the owner) — the PBS datastore lives off-site on Contabo S3 directly; the on-site S3 is not a backup destination; retention locked (§Backup destinations, decisions 1, 6, 7)
**Date:** 2026-04-13 (supersedes flat ADR-0020, 2026-04-02)
**Scope:** Platform-wide backup policy — what gets backed up, where to, how often, how long retained, how to restore. Does not define per-service backup mechanics beyond the class-level pattern.
**Related:** `infra/0003-terraform-standard` (state backup), `security/0001-secret-storage` (Vault Raft snapshots), `infra/0005-environment-tiers §Tier-specific constraints`, `security/0003-hardening §8` (audit log retention)

---

## Context

The platform runs multiple stateful services (PostgreSQL, Redis, GitLab, Vault, NetBox, MinIO, Nextcloud, email). Without a platform-wide backup policy:

- Individual service backup configs diverge or are omitted entirely
- Restore SLA is undefined — recovery time in an incident is unknown until an incident happens
- Off-site protection is unspecified — a single NAS failure could result in data loss
- ISO 27001:2022 A.8.13 and NIS2 Art. 21(2)(c) require documented backup and business continuity controls — compliance evidence needs a written policy, not ad-hoc scripts

This ADR defines the **architectural rules** for backups. Per-service backup configuration lives in each service's `docs/backup.md` runbook.

## Decision

### Per service class

Backups are defined **by service class**, not per individual service. Each service falls into exactly one class, and the class determines schedule, method, retention, and restore SLA.

| Class | Services | Primary method |
|---|---|---|
| **Relational DB** | PostgreSQL (Patroni), MariaDB | `pg_dump` / `mariabackup` + WAL archiving |
| **Cache / queue** | Redis (Sentinel) | `BGSAVE` RDB snapshots |
| **Object store** | MinIO | Bucket replication to secondary location |
| **Secret store** | HashiCorp Vault | `vault operator raft snapshot` |
| **Application** | GitLab CE, NetBox, Authentik, Nextcloud, Mailcow | Application-native backup tool (`gitlab-backup`, `netbox.sh backup`, etc.) |
| **File / volume** | Synology NFS shares, Proxmox VM snapshots | Proxmox Backup Server + NFS snapshot |
| **Infrastructure state** | Terraform state | Handled by `infra/0003-terraform-standard` — not re-covered here |

### Backup destinations

Two-tier destination strategy:

| Phase | Primary | Secondary / off-site |
|---|---|---|
| **2026-04 → 2026-10** | Synology NAS (`br-syno-01`) | PBS datastore on the platform's own S3 (SeaweedFS, on the hypervisor's pool) mirrored to Contabo — the history sat twice on site and once off-site, and the copy on the hypervisor filled its pool (2026-10-18) |
| **Decided 2026-10-20** | Synology NAS (hot): nightly full `vzdump` of every guest, **7 daily + 4 weekly** | **Contabo S3, as the PBS datastore itself** (PBS 4 S3 backend, local cache on the PBS guest, client-side encryption with the key in Vault): **14 daily + 8 weekly + 12 monthly**, deduplicated |

The platform's own S3 (SeaweedFS) is **not a backup destination**: it holds application buckets (registries, artifacts, logs, session recordings) and mirrors those to Contabo. A backup of a guest never lands on the hypervisor that runs it.

Consequences of the off-site datastore: the first fill uploads the deduplicated set once; afterwards only the daily deltas travel. A Contabo outage costs that night's PBS backup, never the NAS one. Restores of recent days come from the NAS (fast, full archives); older points come from Contabo.

**Off-site backup is mandatory for `prod` and `drp` env tiers** (per `infra/0005-environment-tiers`). Non-prod tiers may rely on a single local target.

### 3-2-1 rule

The platform targets the **3-2-1 backup rule** for prod / drp data:
- **3** copies of the data (production + local backup + off-site backup)
- **2** different storage media (disk + object store)
- **1** off-site copy (not in the same physical location as production)

Since 2026-10-20 the platform is at **3-2-1**: production, the NAS (disk, on site) and the PBS datastore on Contabo (object store, off site).

### Encryption

**All backup data at rest is encrypted.**

- Synology NAS: encrypted shared folders with keys stored in HashiCorp Vault at `secret/prod/synology/backup-encryption-key`
- Off-site (once deployed): server-side encryption enabled by default on the chosen provider (SSE-S3 or SSE-KMS)
- Encryption keys never leave Vault unencrypted; `vault kv get -field=key` retrieves them only at backup time

In-transit encryption is TLS on every leg — no plaintext NFS, no unencrypted HTTP to off-site.

### Restore testing

**Restores are the test of backups** — an untested backup is not a backup.

- Restore drills are run on a defined cadence (see §Pending decisions for cadence)
- Every drill produces a report with timing, success / failure, and notes, archived in `platform-setup/runbooks/restore-drills/`
- ISO 27001:2022 A.8.13 compliance requires at least one documented drill per tier per year
- Drill failures trigger an incident ticket, not a retry loop

### Audit logging of backup activity

Backup runs themselves produce logs. These logs are forwarded to Loki per `infra/0006-logging`:

- Start / end timestamps, bytes transferred, success / failure status
- Restore drill events with full detail
- Backup policy violations (skipped schedule, exceeded retention window)

Backup audit logs use the **audit** label class and the 90-day minimum retention from `infra/0006-logging`.

## Pending decisions

The following policy thresholds must be set by @yboujraf before this ADR moves from Draft to Accepted. Until each decision is locked, the corresponding area carries `⚠ TBD`. Partial decisions are better than none — do not block the entire ADR on one unresolved threshold.

| # | Decision | Options / reference | Status |
|---|---|---|---|
| 1 | **Off-site backup target** | **LOCKED 2026-08-30: Contabo S3**; **clarified 2026-10-20:** the PBS datastore is on Contabo directly (not on the on-site S3 with a mirror); Vault snapshots and the application buckets' mirror also go there | ✅ locked |
| 2 | **PostgreSQL backup schedule** | Daily `pg_dump` + continuous WAL archiving? PITR window length? | ⚠ TBD |
| 3 | **Redis backup schedule** | `BGSAVE` hourly / daily? RDB + AOF hybrid? | ⚠ TBD |
| 4 | **Vault Raft snapshot schedule** | **LOCKED 2026-08-30: daily** raft snapshot → NFS + S3 + controller (`roles/vault_backup`) | ✅ locked |
| 5 | **GitLab application backup schedule** | Daily? Delta or full? | ⚠ TBD |
| 6 | **Retention — hot (NAS)** | **LOCKED 2026-10-20: 7 daily + 4 weekly** (full archives, on site) | ✅ locked |
| 7 | **Retention — cold (off-site)** | **LOCKED 2026-10-20: 14 daily + 8 weekly + 12 monthly** on the Contabo PBS datastore (deduplicated, encrypted client-side) | ✅ locked |
| 8 | **Restore SLA (RTO) — prod** | Typical: ≤ 4 hours for critical services, ≤ 24 hours for non-critical | ⚠ TBD |
| 9 | **Restore SLA (RTO) — non-prod** | Typical: best-effort, no SLA | ⚠ TBD |
| 10 | **Restore drill cadence** | Quarterly (ISO 27001 minimum) or monthly? | ⚠ TBD |
| 11 | **Backup encryption KMS** | Vault Transit engine, or per-tool native KMS? | ⚠ TBD |

## Consequences

- **Formal compliance evidence.** ISO 27001:2022 A.8.13 and NIS2 Art. 21(2)(c) are supported by a documented policy + per-service runbooks + drill reports.
- **No stateful service is production-ready without a backup entry.** The deploy checklist for every Layer 5 service includes "backup class assigned, runbook written, first backup verified".
- **Synology NAS is a single point of failure** until off-site is deployed — tracked as a compliance gap in `security/0002-compliance-mapping`.
- **3-2-1 rule is the target, 2-1-0 is the current state** for pre-off-site deployment.
- **Encryption at rest is mandatory** — no unencrypted backup data anywhere.
- **Restore drills are mandatory and timed** — the evidence of backup working is a successful restore, not a completed backup script.
- **Several policy thresholds are pending** (§Pending decisions) — the ADR ships partial and is amended as each decision locks.

## Revision triggers

Revise when:
- Any `⚠ TBD` in §Pending decisions is resolved
- Off-site backup target is chosen and deployed
- A 4th backup destination tier is added (e.g. tape / cold archive)
- Proxmox Backup Server replaces or supplements Synology NAS for VM-level backups
- The 3-2-1 rule is upgraded to 3-2-1-1-0 (one air-gapped, zero errors) or a similar stricter standard
- A compliance framework forces a specific RTO / RPO / retention beyond current targets (e.g. DORA for financial workloads)

## CISO mapping

| Framework | Controls covered |
|---|---|
| ISO 27001:2022 | A.8.13 (information backup — documented policy), A.5.30 (ICT readiness for business continuity — ⚠ partial, drill cadence TBD), A.8.14 (redundancy — ⚠ partial, off-site target TBD) |
| NIS2 | Art. 21(2)(c) (backup management, disaster recovery, crisis management — ⚠ partial pending off-site + drill cadence) |
| GDPR | Art. 32(1)(c) (ability to restore availability — drill-tested restore is the evidence) |

## Locked operational rules (codified 2026-08, `playbooks/pbs-pve-storage.yml`)

- The PBS VM never backs up into its own S3 datastore (circular — its DR copy goes to NFS); excluded by VMID auto-detection.
- OPNsense (FreeBSD) runs with `freeze-fs-on-backup=0` — fs-freeze deadlocks the guest; flag is Ansible-owned, applied via `qm reboot` (a guest reboot does not cycle QEMU).
