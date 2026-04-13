# infra/0007 — Monitoring

**Status:** Draft
**Date:** 2026-04-13 (supersedes flat ADR-0023, 2026-04-02)
**Scope:** Platform-wide metrics collection and alerting — Prometheus + Grafana + Alertmanager. Does not define log aggregation (see `infra/0006-logging`) or tracing.
**Related:** `infra/0006-logging`, `security/0003-hardening §8`, `infra/0008-backup-strategy`, `services/0003-netbox-cmdb` (future)

---

## Context

Platform services span multiple VMs and produce metrics in different formats. A unified monitoring stack is required **before** services are deployed so that observability is built in — not bolted on. Bolt-on monitoring produces the same blind spots as bolt-on logging (see `infra/0006-logging`).

Flat ADR-0017 originally covered both logging and monitoring; flat ADR-0023 split monitoring out as a separate concern. This ADR preserves that split — logging and monitoring are complementary but independently owned.

## Decision

### Stack — Prometheus + Grafana + Alertmanager

| Component | Role | License |
|---|---|---|
| Prometheus | Metrics collection via scrape (default 15s interval) | Apache 2.0 |
| Alertmanager | Alert routing, deduplication, silencing | Apache 2.0 |
| Grafana | Dashboards, alert rules UI, data source aggregation | AGPL v3 |

All three co-located on the observability VM alongside Loki + Promtail (`infra/0006-logging`). One observability host, one lifecycle, one backup policy.

### Scrape requirement — no service goes live without a `/metrics` endpoint

**Every platform service must expose a `/metrics` endpoint** (or equivalent Prometheus exporter). This is a deploy gate — no service is considered production-ready until the scrape target is confirmed.

| Service class | Exporter |
|---|---|
| Go / Python / Rust / Java app | Native Prometheus client library |
| PostgreSQL | `postgres_exporter` |
| Redis | `redis_exporter` |
| MinIO | Native (`/minio/v2/metrics/cluster`) |
| HAProxy / Traefik | Native Prometheus endpoint |
| HashiCorp Vault | Native `/v1/sys/metrics?format=prometheus` |
| OPNsense | `node_exporter` + OPNsense-specific exporter (when available) |
| Linux host | `node_exporter` on every VM |
| Docker containers | `cadvisor` per host |

The **`node_exporter`** is deployed by the Ansible `observability-client` role on every VM, the same role that deploys Promtail. One role, two agents per VM (Promtail + node_exporter).

### Alert baseline — minimum required per service

Every service must define these alerts at minimum:

1. **Service down** — target unreachable for more than 2× scrape interval
2. **Resource saturation** — CPU / memory / disk above a threshold (thresholds pending `security/0003-hardening §Pending decisions`)
3. **Application-specific SLI** — at least one indicator meaningful to the service (e.g. GitLab: `gitlab_rails_queue_duration_seconds`, NetBox: `netbox_queue_backlog`, Vault: `vault_core_unsealed`)

Per-service alerts live in the service's `docs/monitoring.md` and are deployed as Prometheus alerting rules via Ansible.

### Alert routing

Alertmanager routes alerts to destinations based on label:

| Label | Destination |
|---|---|
| `severity=critical` | Discord `#alerts-critical` + email to @yboujraf + PagerDuty (future) |
| `severity=warning` | Discord `#alerts-warning` |
| `severity=info` | Grafana dashboard annotation only, no push |

**Rules:**
- No alert without an action. If there's nothing to do about it, it's a dashboard widget, not an alert.
- Critical alerts have a documented runbook — the alert annotation links to the runbook path
- Silences are time-bound and must specify a reason; silent-forever is forbidden
- Discord webhook token lives in Vault at `secret/prod/alertmanager/discord-webhook` per `security/0001-secret-storage`

### Metrics retention

| Class | Retention |
|---|---|
| Standard metrics (host, container, app) | 90 days |
| Security-relevant metrics (Vault seal status, Authentik login failures, firewall drops) | 180 days |

Retention enforced by Prometheus TSDB with the `--storage.tsdb.retention.time` flag; long-term archival to Contabo S3 via Prometheus remote-write (future) for metrics that need 1+ year retention.

### Grafana dashboards

**Dashboards are provisioned as code**, not clicked together in the UI.

- Dashboard JSON lives in `ansible-platform/roles/grafana/files/dashboards/` and is loaded via Grafana file provisioning
- Every service has at least one dashboard with tags `service:{name}`, `env:{env}`, `scope:{scope}`
- Dashboard links in per-service `docs/monitoring.md` use the dashboard UID (stable across renames), not the title
- Manual changes in the Grafana UI are **break-glass only** — reconciled back to git within one day

### Observability URL registry pattern

Grafana is the **live observability registry**. Rather than hardcoding dashboard URLs in documentation and code:

- NetBox custom field `prometheus_job` links each service record to its metrics job label (per `services/0003-netbox-cmdb` when refactored)
- Agents (Rune, future observability agents) query the Grafana API for dashboards tagged with a given `service` / `env` pair — no static URLs
- This keeps documentation from going stale when a dashboard is renamed or moved

### Zabbix — explicitly not in scope

Zabbix is **not** part of the active PoC stack. It is a Phase 2 candidate only, and only if SNMP polling of legacy network devices is needed at a scale where Prometheus SNMP exporters become insufficient. **Do not deploy Zabbix in PoC.** If you think you need it, open a RAID item first.

## Consequences

- **Observability VM is a Layer 5 prerequisite** (shared with `infra/0006-logging`) — no service is deployed before Prometheus + Grafana are running and scrape targets are confirmed
- **Every VM runs `node_exporter`** — delivered by the Ansible `observability-client` role, same role that installs Promtail
- **Every service has a `/metrics` endpoint** — this is a deploy gate, not optional
- **Every service has at least one Grafana dashboard** and at least one alert — also a deploy gate
- **Alerting is noise-free** — no alert without an action, runbooks linked, silences justified
- **Dashboards are version-controlled** — no click-to-build, no one-off changes surviving beyond a day
- **Zabbix is reserved for a future decision** — not active today, not deployed speculatively

## Revision triggers

Revise when:
- Prometheus federation is needed (multi-site, multi-cluster) — adds a top-level aggregation layer
- VictoriaMetrics or Mimir replaces Prometheus for higher retention / cardinality
- Zabbix is actually needed for a specific device class Prometheus cannot cover
- Distributed tracing (Tempo, Jaeger, OpenTelemetry) is added — expand the stack to PLGT
- PagerDuty or similar paging service is integrated (currently Discord + email only)
- A compliance framework forces a specific metrics retention window beyond the 180-day security class

## CISO mapping

| Framework | Controls covered |
|---|---|
| ISO 27001:2022 | A.8.16 (monitoring activities — Prometheus + Grafana + alert baseline), A.8.15 (logging — separated to `infra/0006-logging` but monitoring contributes to event logging via audit metrics) |
| NIS2 | Art. 21(2)(b) (incident handling — alerts + runbooks + routing), Art. 21(2)(a) (risk management — saturation and SLI alerts catch problems before outages) |
