<!--
  SCOPE GUARD — INFRA ADR
  ========================
  This template is for infrastructure decisions only.
  DO NOT use this template for:
  - Library implementation details (DI, test strategy, log format)
  - Developer tooling (devcontainer, pre-commit hooks)
  - App-level credential handling
  Wrong template = PR blocked.
-->

# ADR-0020: Platform backup strategy

**Status:** Draft
**Date:** 2026-04-02
**Deciders:** @yboujraf

---

## Context

The BY-SYSTEMS PoC platform runs multiple stateful services (PostgreSQL, Redis, GitLab, Vault, NetBox, MinIO, Nextcloud). No platform-wide backup policy has been formally defined. Without it:

- Individual service backup configs diverge or are omitted
- Restore SLA is undefined — recovery time in an incident is unknown
- Off-site protection is unspecified — single NAS failure could result in data loss
- ISO 27001:2022 A.8.13 and NIS2 Art. 21(2)(c) require documented backup and business continuity controls

This ADR defines the platform backup standard. Per-service backup configuration is documented in each tool's `docs/backup.md`.

## Decision

**Backup policy is defined per service class.** The platform backup standard (`docs/standards/backup-standard.md`) is the governing document. This ADR captures the architectural decisions behind it.

### Backup backend

| Phase | Backend | Notes |
|---|---|---|
| PoC | Synology NAS (`/by-terraform-state/` and per-service paths) | Already in use for Terraform state |
| Phase 2 | [OWNER TO DEFINE: off-site S3 target — Contabo S3 / Backblaze B2 / other] | Off-site copy required for production |

### Schedule

| Service class | Schedule | Method |
|---|---|---|
| PostgreSQL | [OWNER TO DEFINE: daily `pg_dump` + continuous WAL archiving?] | `pg_dump` + WAL to NAS |
| Redis | [OWNER TO DEFINE: RDB snapshot schedule?] | `BGSAVE` triggered by cron |
| GitLab | [OWNER TO DEFINE: daily application backup?] | GitLab backup rake task |
| Vault | [OWNER TO DEFINE: Raft snapshot schedule?] | `vault operator raft snapshot` |
| MinIO | [OWNER TO DEFINE: bucket replication schedule?] | MinIO client mirror |
| Nextcloud | [OWNER TO DEFINE: daily?] | Volume snapshot + database dump |

### Retention

| Tier | Hot retention | Cold retention |
|---|---|---|
| PoC | [OWNER TO DEFINE: e.g., 7 days on NAS] | [OWNER TO DEFINE: e.g., 30 days in S3] |
| Production | [OWNER TO DEFINE] | [OWNER TO DEFINE] |

### Restore SLA (RTO)

| Tier | RTO |
|---|---|
| PoC | [OWNER TO DEFINE: e.g., best-effort, no SLA] |
| Production | [OWNER TO DEFINE: e.g., < 4 hours for critical services] |

### Encryption

Backup encryption at rest: [OWNER TO DEFINE: required in prod? encryption standard? key management?]

### Off-site copy

Off-site backup copy: [OWNER TO DEFINE: required for prod? target?]

### Restore testing

Restore drill cadence: [OWNER TO DEFINE: quarterly recommended for ISO 27001 compliance]

## VM / Resource Spec

Not applicable — backup strategy uses existing infrastructure.

## Network

Not applicable — backup traffic uses OOB/MGMT network (existing).

## Storage

| Component | Backend | Volume / Path | Backup policy |
|---|---|---|---|
| Terraform state | Synology NAS | `/by-terraform-state/{env}/` | Synced after each `apply`/`destroy` |
| PostgreSQL | Synology NAS | [OWNER TO DEFINE: NAS path] | [OWNER TO DEFINE: schedule] |
| Redis | Synology NAS | [OWNER TO DEFINE: NAS path] | [OWNER TO DEFINE: schedule] |
| GitLab | Synology NAS | [OWNER TO DEFINE: NAS path] | [OWNER TO DEFINE: schedule] |
| Vault | Synology NAS | [OWNER TO DEFINE: NAS path] | [OWNER TO DEFINE: schedule] |

## TLS / PKI

Not applicable — backup paths are internal, NAS access over OOB/MGMT network.

## CISO mapping

> Applies only to controls directly relevant to this ADR's scope.

### ISO/IEC 27001:2022

| Control | Title | Status | Notes |
|---|---|---|---|
| A.8.13 | Information backup | ⚠ Partial | Policy defined; per-service implementation pending |
| A.8.14 | Redundancy of information processing facilities | ✗ Gap | [OWNER TO DEFINE: remediation plan — HA for critical services] |
| A.5.30 | ICT readiness for business continuity | ⚠ Partial | RTO defined as placeholder; restore drills not yet scheduled |

### NIS2 (Directive 2022/2555)

| Article | Requirement | Status | Notes |
|---|---|---|---|
| Art. 21(2)(c) | Backup management, disaster recovery, crisis management | ⚠ Partial | Backup policy drafted; off-site and drill cadence pending @yboujraf sign-off |

### GDPR (Regulation 2016/679)

| Article | Requirement | Status | Notes |
|---|---|---|---|
| Art. 32(1)(c) | Ability to restore availability of personal data after incident | ⚠ Partial | Depends on per-service backup completion — services processing personal data not yet all covered |

## Licensing

No new tools introduced by this ADR — backup uses existing platform tooling.

## Consequences

**Enables:**
- Formal ISO 27001:2022 A.8.13 and NIS2 Art. 21(2)(c) compliance evidence
- Consistent backup coverage across all stateful services
- Defined restore SLA for incident response planning

**Constrains:**
- Each stateful service must implement backup before being considered production-ready
- Synology NAS is a single point of failure in PoC — off-site copy required for production

**Known risks:**
- RTO and retention thresholds are placeholders — incident response is undefined until @yboujraf sets values
- No restore drill scheduled — compliance evidence for ISO 27001 will require at least one documented drill
- Off-site copy not yet defined — NAS failure in PoC results in backup loss

## References

- `docs/standards/backup-standard.md` — platform backup standard (governing document)
- [ISO 27001:2022 A.8.13](https://www.iso.org/standard/27001)
- [NIS2 Art. 21(2)(c)](https://eur-lex.europa.eu/legal-content/EN/TXT/?uri=CELEX%3A32022L2555)
- [Synology NAS documentation](https://www.synology.com/en-global/support)
