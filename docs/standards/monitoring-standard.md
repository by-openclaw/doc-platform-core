<!--
| Field        | Value            |
|--------------|------------------|
| Created      | 2026-04-02       |
| Last updated | 2026-04-02       |
| Updated by   | Opus             |
-->

# Monitoring Standard

**Status:** Accepted
**Date:** 2026-04-02
**Owner:** @yboujraf
**ADR:** ADR-0017

## Rule

Every platform service must expose a Prometheus metrics endpoint and must be instrumented with a minimum alert baseline. Metrics must be retained for a minimum of 90 days. Services without native Prometheus support must use an exporter.

## Requirements

1. Every service MUST expose a Prometheus-compatible `/metrics` endpoint (native or via exporter).
2. The Prometheus scrape interval MUST be 15 seconds (platform standard).
3. Metrics retention MUST be a minimum of 90 days.
4. Every service MUST have a minimum alert baseline covering: service-down, high CPU, high memory, high disk usage.
5. Alert thresholds: [OWNER TO DEFINE per service — fill in `docs/monitoring.md` per tool].
6. Every service `docs/monitoring.md` MUST document: metrics endpoint URL, scrape job name, expected labels, and Grafana dashboard reference.
7. Grafana dashboard links remain `TODO: create after Grafana deployed` until the service is live — this is acceptable during initial setup.
8. `vm-observability-poc-01` must be provisioned before any other service goes live.
9. Alertmanager MUST be configured with at minimum one notification channel before any service is considered production-ready.

## Standard Prometheus scrape config shape

```yaml
- job_name: '{service}'
  static_configs:
    - targets: ['{vm-ip}:{metrics-port}']
  labels:
    env: '{env}'
    service: '{service}'
```

## Compliance table

| Requirement | Test | Pass condition |
|---|---|---|
| Metrics endpoint reachable | `curl http://{vm-ip}:{port}/metrics` | HTTP 200, `text/plain; version=0.0.4` |
| 15s scrape interval | Prometheus `prometheus.yml` | `scrape_interval: 15s` global or per-job |
| 90-day retention | Prometheus `--storage.tsdb.retention.time` flag | `2160h` or greater |
| Alert baseline defined | `docs/monitoring.md` for each tool | All four baseline alerts documented |
| Grafana dashboard tagged | Grafana API | Dashboard with `service:{name}` and `env:{env}` tags exists |

## Override procedure

To override this standard for a specific tool:
1. Create `tools/{tool}/docs/override-monitoring.md`
2. State: What is different / Why / Compensating control / Reviewed by @yboujraf
3. PR must include override doc before merge is allowed

## References

- [Prometheus documentation](https://prometheus.io/docs/)
- [Prometheus exporters list](https://prometheus.io/docs/instrumenting/exporters/)
- [Grafana documentation](https://grafana.com/docs/grafana/latest/)
- [Alertmanager documentation](https://prometheus.io/docs/alerting/latest/alertmanager/)
