# infra/0006 — Logging

**Status:** Draft
> ⚠ **Implementation gap (verified 2026-08-30):** Loki runs (`lxc-monitoring-01`, SeaweedFS backend) but **Promtail is not deployed on the fleet** — nothing ships yet, including the auditd trail `security/0003 §8` depends on. Shipping lands with the monitoring workstream.
**Date:** 2026-04-13 (supersedes flat ADR-0017, 2026-04-02)
**Scope:** Platform-wide log aggregation — Loki + Promtail, object storage backend, retention, and the no-PII rule. Does not define monitoring (metrics), audit logging policy beyond retention, or per-service log format.
**Related:** `infra/0007-monitoring`, `security/0001-secret-storage` (Vault audit log target), `security/0003-hardening §8` (audit logging requirement), `infra/0008-backup-strategy`

---

## Context

Platform services span multiple VMs and produce logs in different formats (Docker JSON, syslog, application-specific). A unified log aggregation approach is required **before** services are deployed so that log collection is built in — not bolted on. Bolt-on logging creates blind spots during the critical first weeks of any deployment.

## Decision

### Stack — Loki + Promtail

| Component | Role | License |
|---|---|---|
| Loki | Log aggregation and storage | AGPL v3 |
| Promtail | Log shipping agent, one per host | AGPL v3 |

Both run on the observability VM per `infra/0007-monitoring`. Loki and Prometheus are co-located because they share the same observability lifecycle (deploy, scale, backup together).

### Log shipping

**Every host runs a Promtail agent.** No exceptions.

- Promtail is installed and configured by the Ansible `observability-client` role (future — deployed as part of Layer 5 per `infra/0002-platform-charter`)
- Promtail tails `/var/log/*` and Docker container JSON logs
- Syslog (RFC 5424) is supported for system-level components that cannot emit structured logs directly
- Promtail adds labels: `host`, `service`, `env`, `scope` — all mandatory, derived from NetBox host metadata

### Storage backend

Loki uses **S3-compatible object storage** as the backend. Two deployment modes:

| Phase | Backend | Location |
|---|---|---|
| **Current / small scale** | SeaweedFS on-prem (see `services/0009-object-storage`) | `lxc-seaweedfs-01` in SVC zone |
| **Larger scale** | Contabo S3 | Off-site, encrypted at rest |

Boundary between the two is sized by log volume and retention — the platform may run both (SeaweedFS for recent logs, Contabo for cold archive). Boundary details are operational, not architectural.

### Retention

- **Default retention:** 30 days, minimum
- **Audit-relevant services** (Vault, Authentik, GitLab, OPNsense): **90 days minimum** to match NIS2 Art. 23 incident notification windows
- Retention is enforced by Loki compactor / retention config — per-stream overrides allowed via Loki config, not via Promtail labels

### No-PII, no-credentials rule

**No credentials, no personal data, no secrets in any log stream.** This is a hard rule, not a best-effort guideline.

**Enforced at the application level, not by a Loki filter:**
- Structured logs use explicit field allowlists — fields not on the list are not emitted
- Application loggers (Python `structlog`, Go `slog`, etc.) mask known-secret fields by name (`password`, `token`, `secret`, `key`, `authorization`) at the emitter
- Pre-commit hooks catch hardcoded secrets in log statements (`detect-secrets`, `gitleaks`)
- Loki does **not** scrub logs in flight — it receives only what the application emits. If a secret reaches Loki, it is a violation and the service's logger config is the bug, not Loki's.

**Specific prohibitions:**
- No full request / response bodies
- No cookies, bearer tokens, session IDs
- No email addresses, phone numbers, physical addresses, IDs in plaintext
- No SSH keys, GPG keys, TLS private keys in any form
- No full SQL query strings with literal values (parameterize + log the template)

### Structured logging

**All services must emit structured JSON logs.** Free-text logs are acceptable **only** for system-level components (kernel, sshd, fail2ban, etc.) that cannot be configured to emit JSON.

Standard fields (mandatory when available):

| Field | Meaning |
|---|---|
| `timestamp` | ISO 8601 with timezone |
| `level` | `debug` / `info` / `warn` / `error` / `fatal` |
| `service` | Service short code (matches `naming/0001-infra §5`) |
| `env` | Environment tier per `infra/0005-environment-tiers` |
| `host` | Short hostname (matches NetBox `name`) |
| `event` | Short machine-readable event name |
| `message` | Human-readable message |
| `trace_id` / `span_id` | When distributed tracing is in use |

### Audit log destinations

Certain security-relevant log streams have Loki as their **mandatory** destination, not a convenience:

- Vault audit log (every `kv read` / `write` / policy change) — per `security/0001-secret-storage`
- sshd session activity (login, logout, command trail via PAM) — per `identity/0004-os-accounts`
- fail2ban ban / unban events
- OPNsense firewall logs for blocked traffic
- Traefik access logs (with PII redaction applied by Traefik before emission)

These streams carry the `audit` label and use the 90-day retention class.

### Dashboards and alerting

Alerting on log patterns is **not** in this ADR — it lives in `infra/0007-monitoring` (Grafana + Prometheus + Alertmanager). Loki feeds Grafana dashboards for log exploration; alerts are expressed as LogQL queries in Grafana alert rules.

## Consequences

- **Observability VM is a Layer 5 prerequisite.** No other Layer 5 service goes live before `vm-loki-01` (or the combined observability VM) is running and scrape targets are confirmed.
- **Every VM runs Promtail** — rooted in the Ansible `observability-client` role, installed by default in every Proxmox cloud-init template.
- **Every service emits structured JSON.** Services that can't (legacy components) use syslog, which is an exception requiring justification in the service's `docs/monitoring.md`.
- **Secrets in logs are a hard violation, not a soft warning.** Caught via application-level masking, pre-commit hooks, and manual review during PR — never via Loki-side scrubbing.
- **Audit logs and normal application logs share one stack.** Differentiation is by label and retention class, not by separate infrastructure.
- **No ELK, no Splunk, no commercial log management.** Loki + SeaweedFS S3 is the only supported stack.

## Revision triggers

Revise when:
- Log volume exceeds Loki's comfortable scale on a single VM (currently ~100 GB/day is the soft ceiling; if exceeded, split compactor / querier / ingester components)
- Loki is replaced (e.g. by Grafana Tempo integration if tracing overtakes log-centric observability)
- MinIO on-prem is deprecated in favor of Contabo S3 as the sole backend (or vice versa)
- A new compliance framework requires separate audit log infrastructure with dedicated retention (e.g. PCI-DSS 1-year requirement)
- Structured logging field list changes — add / remove mandatory fields
- A service-specific PII redaction requirement forces Loki-side scrubbing (currently forbidden)

## CISO mapping

| Framework | Controls covered |
|---|---|
| ISO 27001:2022 | A.8.15 (logging — centralized stack, mandatory Promtail coverage), A.5.33 (protection of records — retention defined, audit class separated) |
| NIS2 | Art. 21(2)(b) (incident handling — log centralization supports investigation), Art. 23 (incident notification — 90-day audit retention matches reporting windows) |
| GDPR | Art. 32(1)(b) (confidentiality, integrity, availability — no PII / credentials in logs enforced at source) |
