# ADR-0023: Monitoring Approach

**Status:** Accepted
**Date:** 2026-04-02
**Deciders:** @yboujraf

---

## Context

Platform services span multiple VMs and produce metrics in different formats. A unified monitoring stack is required before services are deployed so that observability is built in — not bolted on. ADR-0017 originally covered both logging and monitoring; this ADR splits out the monitoring decision as a separate concern.

## Decision

**Monitoring stack:** Prometheus + Grafana, colocated on `vm-observability-poc-01`.

| Component | Role |
|---|---|
| Prometheus | Metrics collection via scrape (15s interval) |
| Grafana | Dashboards and alerting UI |

**Scrape requirement:** Every platform service must expose a `/metrics` endpoint (or equivalent exporter). This is a deploy gate — no service goes live without a confirmed scrape target.

**Alert baseline:** Each service must define a minimum alert set:
- Service down (target unreachable)
- Resource saturation (CPU, memory, disk — thresholds: `[OWNER TO DEFINE]`)
- Application-specific SLI (defined per service in `docs/monitoring.md`)

**Metrics retention:** `[OWNER TO DEFINE]` (suggested: 90 days on Contabo S3)

**Grafana dashboards:** Provisioned as code via `tools/grafana/config/`. Dashboard links in per-tool `docs/monitoring.md` remain `TODO` until Grafana is deployed.

**Zabbix:** Not in the active PoC stack. Phase 2 candidate only. Do not deploy Zabbix in PoC.

## Consequences

- `vm-observability-poc-01` must be provisioned before other services go live.
- Cloud-init or Ansible role deploys the Prometheus node exporter on every VM.
- Every tool folder must contain `docs/monitoring.md` with scrape endpoint and alert definitions.
- Grafana dashboard provisioning is a Layer 5 task — deferred until Grafana is deployed.

## CISO mapping

### ISO/IEC 27001:2022

| Control | Title | Status | Notes |
|---|---|---|---|
| A.8.16 | Monitoring activities | ⚠ Partial | Prometheus + Grafana defined; per-service alert baselines not yet written |
| A.8.15 | Logging | ✓ Covered | Logging is ADR-0017; monitoring is this ADR — clearly separated |

### NIS2 (Directive 2022/2555)

| Article | Requirement | Status | Notes |
|---|---|---|---|
| Art. 21(2)(b) | Incident handling | ⚠ Partial | Monitoring stack enables detection; alert baselines per-service still required |

### GDPR (Regulation 2016/679)

Not applicable — monitoring covers infrastructure metrics only, no personal data.

## References

- ADR-0017 — Logging Standard (scope: log aggregation only; this ADR covers metrics)
- ADR-0006 — Platform Charter (Layer 5: Platform Services)
- `docs/stack.md` — Prometheus, Grafana, Zabbix entries
- `docs/standards/monitoring-standard.md` — operational rule implementing this decision
