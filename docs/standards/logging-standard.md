<!--
| Field        | Value            |
|--------------|------------------|
| Created      | 2026-04-02       |
| Last updated | 2026-04-02       |
| Updated by   | Opus             |
-->

# Logging Standard

**Status:** Accepted
**Date:** 2026-04-02
**Owner:** @yboujraf
**ADR:** ADR-0017

## Rule

All platform services must emit structured logs to stdout, ship them to Loki via Promtail, and retain them for a minimum of 30 days. No credentials or PII may appear in any log line.

## Requirements

1. All services MUST emit logs in a structured format (JSON or syslog RFC5424).
2. All VMs MUST run a Promtail agent that ships logs to the central Loki instance.
3. Log retention MUST be a minimum of 30 days in Loki.
4. NO credentials, secrets, or tokens may appear in any log line — ever.
5. NO personal data (PII) may appear in any log line unless explicitly required and with data masking applied.
6. Loki log storage backend MUST be S3-compatible (Contabo S3 in PoC).
7. Every service `docs/monitoring.md` MUST document the expected Loki labels and log format.
8. Log shipping failure MUST trigger an alert — silent log gaps are not acceptable.
9. Docker container logs MUST use the JSON file driver (default) — Promtail reads them automatically.

## Standard Loki label set

Every log stream shipped to Loki MUST include the following labels:

| Label | Example value | Description |
|---|---|---|
| `host` | `vm-authentik-poc-01` | VM hostname |
| `env` | `poc` | Environment tier |
| `service` | `authentik` | Service name |
| `component` | `worker` | Sub-component within the service |

## Compliance table

| Requirement | Test | Pass condition |
|---|---|---|
| Structured log format | Inspect log samples from Loki | JSON or syslog RFC5424 — no plain text blobs |
| Promtail running | `systemctl is-active promtail` on each VM | `active` |
| 30-day retention | Loki retention config | `retention_period: 720h` or greater |
| No credentials in logs | Grep logs for known secret patterns | Zero matches |
| S3 backend configured | Loki config `storage_config` section | S3-compatible backend configured |
| Labels present | Query Loki for `{host=~".+", env=~".+", service=~".+"}` | All streams have required labels |

## Override procedure

To override this standard for a specific tool:
1. Create `tools/{tool}/docs/override-logging.md`
2. State: What is different / Why / Compensating control / Reviewed by @yboujraf
3. PR must include override doc before merge is allowed

## References

- [Loki documentation](https://grafana.com/docs/loki/latest/)
- [Promtail configuration](https://grafana.com/docs/loki/latest/send-data/promtail/)
- [RFC 5424 — syslog](https://tools.ietf.org/html/rfc5424)
- [GDPR Art. 32 — Security of processing](https://gdpr.eu/article-32-security-of-processing/)
