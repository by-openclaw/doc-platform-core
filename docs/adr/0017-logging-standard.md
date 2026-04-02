# ADR-0017: Logging Standard

- **Status:** Draft
- **Date:** 2026-04-02
- **Deciders:** @yboujraf

## Context

All platform services produce logs in different formats and destinations. A unified logging standard is needed to ensure consistent log collection via Promtail, aggregation in Loki, and querying in Grafana — covering both Docker container JSON logs and bare-metal/VM syslog output.

## Decision

TODO: pending @yboujraf review

## Consequences

TODO: pending decision

## References

- `docs/stack.md` — Loki, Promtail, Grafana, Zabbix entries
- ADR-0006 — Layer 5 (PoC Services) observability requirement
- `brainstorming/2026-04-02-doc-matrix.md` — missing ADR identified here
</content>