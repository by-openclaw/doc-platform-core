# ADR-0017: Logging Standard

**Status:** Accepted
**Date:** 2026-04-02
**Deciders:** @yboujraf
**Revised:** 2026-04-02 — scope tightened to logging only. Monitoring decision moved to ADR-0023.

---

## Context

Platform services span multiple VMs and produce logs in different formats (Docker JSON, syslog, application-specific). A unified log aggregation approach is required before services are deployed so that log collection is built in — not bolted on.

## Decision

**Log aggregation stack:** Loki + Promtail, colocated on `vm-observability-poc-01`.

| Component | Role |
|---|---|
| Loki | Log aggregation and storage |
| Promtail | Log shipping agent on each VM |

**Log shipping:** Promtail agents ship logs to Loki. Docker container JSON logs and syslog are both supported.

**Storage backend:** Loki uses S3-compatible object storage on Contabo as the backend.

**Log retention:** 30 days (minimum). `[OWNER TO DEFINE: longer retention for audit-relevant services?]`

**No-PII rule:** No credentials, no personal data, no secrets in logs. Enforced at the application level — not a Loki filter.

**Structured logging:** All services must emit structured JSON logs. Free-text syslog is acceptable for system-level components only.

> **Monitoring (Prometheus + Grafana) is a separate concern.** See ADR-0023 — Monitoring Approach.

## Consequences

- `vm-observability-poc-01` must be provisioned before other services go live (no logging blind spot during initial deployment).
- Every VM must run a Promtail agent. Cloud-init or Ansible role deploys it.
- Service `docs/monitoring.md` files must document expected Loki labels and log format.
- No credentials or PII may appear in any log shipped to Loki — this is a hard rule, not a best-effort guideline.

## CISO mapping

### ISO/IEC 27001:2022

| Control | Title | Status | Notes |
|---|---|---|---|
| A.8.15 | Logging | ⚠ Partial | Logging stack defined; per-service Promtail coverage not yet complete |
| A.5.33 | Protection of records | ✓ Covered | Loki with S3 backend; 30-day retention defined |

### NIS2 (Directive 2022/2555)

| Article | Requirement | Status | Notes |
|---|---|---|---|
| Art. 21(2)(b) | Incident handling | ⚠ Partial | Log centralisation supports incident investigation; alert baselines in ADR-0023 |

### GDPR (Regulation 2016/679)

| Article | Requirement | Status | Notes |
|---|---|---|---|
| Art. 32(1)(b) | Confidentiality, integrity, availability | ✓ Covered | No PII/credentials in logs enforced; log retention defined |

## References

- ADR-0023 — Monitoring Approach (Prometheus + Grafana — separate ADR)
- ADR-0006 — Platform Charter (Layer 5: Platform Services)
- `docs/stack.md` — Loki, Promtail entries
- `docs/standards/logging-standard.md` — operational rule implementing this decision
