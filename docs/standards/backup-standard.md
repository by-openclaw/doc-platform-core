<!--
| Field        | Value            |
|--------------|------------------|
| Created      | 2026-04-02       |
| Last updated | 2026-04-02       |
| Updated by   | Opus             |
-->

# Backup Standard

**Status:** Draft
**Date:** 2026-04-02
**Owner:** @yboujraf
**ADR:** ADR-0020 (stub — pending @yboujraf sign-off)

## Rule

Every stateful platform service must have a documented and tested backup procedure. Backup coverage, schedule, retention, backend, and restore SLA must be explicitly defined before a service is considered production-ready.

## Requirements

1. Every stateful service MUST have a documented backup procedure in `tools/{tool}/docs/backup.md`.
2. Backup schedule: [OWNER TO DEFINE: daily / weekly / continuous WAL — specify per service class].
3. Retention: [OWNER TO DEFINE: minimum retention period in days — separate hot/cold tiers if applicable].
4. Backup backend: Synology NAS (PoC) → [OWNER TO DEFINE: off-site / S3 target for Phase 2].
5. Off-site copy: [OWNER TO DEFINE: requirement — yes/no, target location, encryption standard].
6. Restore SLA: [OWNER TO DEFINE: RTO per tier — poc vs prod].
7. Backup encryption: [OWNER TO DEFINE: at-rest encryption required? key management?].
8. Backup testing: [OWNER TO DEFINE: frequency of restore drills — quarterly recommended for ISO 27001].
9. Backups MUST be verified after each run (checksum or test restore). Silent backup failures are not acceptable.
10. Services without durable state (e.g., pure cache) MUST explicitly document "no backup required" and state why.

## Service backup classes

| Class | Examples | Typical method |
|---|---|---|
| Database | PostgreSQL, Redis | `pg_dump` + WAL archiving / RDB snapshot |
| Object storage | MinIO | Bucket replication to NAS / off-site S3 |
| Application data | GitLab repos, Vault secrets | Volume snapshot + export |
| Config | Traefik, Prometheus configs | Git (these live in the repo) |

## Compliance table

| Requirement | Test | Pass condition |
|---|---|---|
| Backup doc exists | `ls tools/{tool}/docs/backup.md` | File exists and is non-empty |
| Backup schedule configured | Inspect backup script / cron | Schedule matches defined policy |
| Retention configured | Backup config | Retention >= defined minimum |
| Backup verification | Backup run logs | No silent failures in last 30 days |
| Restore tested | Runbook or test record | Restore completed successfully within defined RTO |

## Override procedure

To override this standard for a specific tool:
1. Create `tools/{tool}/docs/override-backup.md`
2. State: What is different / Why / Compensating control / Reviewed by @yboujraf
3. PR must include override doc before merge is allowed

## References

- [ISO 27001:2022 A.8.13 — Information backup](https://www.iso.org/standard/27001)
- [NIS2 Art. 21(2)(c) — Business continuity and backup management](https://eur-lex.europa.eu/legal-content/EN/TXT/?uri=CELEX%3A32022L2555)
- [Synology NAS documentation](https://www.synology.com/en-global/support)
