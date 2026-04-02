# ADR-0017: Logging Standard

- **Status:** Accepted
- **Date:** 2026-04-02
- **Deciders:** @yboujraf

## Context

Platform services span multiple VMs and produce logs in different formats (Docker JSON, syslog, application-specific). A unified observability stack is required to ensure logs and metrics are collected, retained, and queryable in a single place. The choice of stack must be decided before services are deployed so that log collection is built in — not bolted on.

## Decision

**Observability stack:** Prometheus + Grafana + Loki, colocated on `vm-observability-poc-01` (`10.1.3.50`).

| Component | Role |
|---|---|
| Prometheus | Metrics collection via scrape (15s interval) |
| Loki | Log aggregation |
| Grafana | Dashboards and alerting UI |

**Log shipping:** Promtail agents on each VM ship logs to Loki. Docker container JSON logs and syslog are both supported.

**Storage backend:** Loki uses S3-compatible object storage on Contabo as the backend.

**Retention:**
- Logs: 30 days
- Metrics: 90 days

**Zabbix:** Not in the active PoC stack. Phase 2 candidate only. Do not deploy Zabbix in PoC.

## Consequences

- `vm-observability-poc-01` must be provisioned before other services go live (no monitoring blind spot during initial deployment).
- Every VM must run a Promtail agent. Cloud-init or Ansible role deploys it.
- Service `docs/monitoring.md` files must document Prometheus scrape endpoint and expected Loki labels.
- Grafana dashboard links remain `TODO: create after Grafana deployed` until the service is live.
- Zabbix tooling in `platform-setup` is scaffolded but not activated in PoC configs.

## References

- `docs/stack.md` — Loki, Promtail, Grafana, Zabbix entries
- ADR-0006 — Layer 5 (PoC Services) observability requirement
- `brainstorming/2026-04-02-opus-batch1-feedback.md` — Grafana dashboard TODO confirmed acceptable
