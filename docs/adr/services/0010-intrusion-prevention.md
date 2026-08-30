# services/0010 — Intrusion Prevention (CrowdSec)

**Status:** Draft (documents the deployed architecture)
**Date:** 2026-08-30
**Scope:** Platform IPS — detection engine, decision distribution, enforcement points, enrolment lifecycle. Closes the review gap: IPS was core infrastructure with no owning ADR.
**Related:** `security/0003-hardening`, `services/0001-opnsense`, `services/0006-firewall-services`, `infra/0006-logging`

---

## Context

The platform needs coordinated intrusion prevention across the edge firewall, the TLS edge, and every guest. Suricata-style inline IPS was evaluated and **not** chosen as the engine: CrowdSec's behaviour-based detection + shared decision model fits a many-small-hosts platform better; Suricata survives only as a WAN signature feed.

## Decision

### 1. Engine — CrowdSec, hub-and-spoke

- **Central LAPI** on `lxc-crowdsec-01` — the single decision database.
- **Agents (log processors) on every guest** — parse local logs (sshd, service logs), push alerts to the LAPI. Enrolment via the platform registration token (`crowdsec/registration-token` in Vault).
- **Bouncers enforce**: the **OPNsense firewall bouncer** (blocks at the edge, both directions) and the **Traefik bouncer** (blocks at the TLS edge). Enforcement is central — an attack seen by one host is blocked for all.
- **Suricata WAN feed**: signature detection on the WAN interface feeds CrowdSec scenarios; no inline Suricata IPS.
- Console enrolment (SaaS visibility) is optional per deployment (`crowdsec/console*` secrets).

### 2. Enrolment lifecycle

- New guest → agent installed + enrolled by the baseline (`playbooks/crowdsec-agents.yml`, automatic for inventory members).
- **Rebuild rule:** a rebuilt guest leaves a stale machine registration — **delete the old machine at the LAPI before re-enrolling** (a `lapi status` rc=0 can be a false positive; verify the machine list).
- Decommission: the machine entry is removed by the `service_decommission` catalog.

### 3. Secrets

Per `security/0001`: bouncer keys and tokens live in Vault under `prod/crowdsec/*` (fabric working copy).

## Consequences

- One decision plane; enforcement at the two choke points (FW + Traefik) rather than per-host firewalls.
- Agents are cheap (log parsing); no inline packet inspection on guests.
- LAPI is a single point of decision distribution — its guest is part of the core batch (identity-hardened, backed up); enforcement fails OPEN (bouncers without decisions block nothing) — acceptable: base FW policy still applies.

## Revision triggers

Engine replacement; a second enforcement point class (e.g. per-guest bouncers); LAPI HA; inline IPS requirement returning.

## CISO mapping

| Framework | Controls |
|---|---|
| ISO 27001:2022 | A.8.16 (monitoring), A.8.20/A.8.21 (network security), A.5.7 (threat intelligence — hub scenarios) |
| NIS2 | Art. 21(2)(b) (incident handling), 21(2)(e) (systems security) |
