# services/0006 — Firewall Services (IDS/IPS + Internet-by-Request)

**Status:** Draft
**Date:** 2026-05-23
**Scope:** OPNsense services not covered by `services/0001-opnsense` (provisioning) or `naming/0003-firewall` (aliases/rules naming): Suricata IDS/IPS, time-limited per-host internet bypass.
**Related:** `services/0001-opnsense`, `naming/0003-firewall`, `infra/0004-network-architecture`, `infra/0006-logging`, `security/0001-secret-storage`

---

## Context

Two service-level needs are not covered by existing ADRs:

1. **Intrusion detection** on the WAN edge. The platform exposes services (DMZ, VPN) and needs at minimum an alert path for known attack signatures.
2. **Temporary internet egress** for hosts that are normally default-deny outbound (Storage, CCTV, some SVC hosts during license activation / OS upgrade). The current pattern is to edit the firewall manually per incident — does not scale, leaves stale rules.

This ADR defines both as first-class services with explicit ownership and contracts.

## Decision

### 1. Suricata IDS — alert-only on WAN1

- Plugin: `os-intrusion-detection` (Suricata 7.x), installed via Ansible (`ansible-platform/roles/opnsense/tasks/ids.yml`)
- Interface: WAN1 only (PPPoE / `pppoe0`). WAN2 added when active.
- Mode: **IDS (alert)** at initial deploy. **Not** IPS (drop) — see promotion gate §2.
- Rulesets:
  - **ET Open** (Emerging Threats free) — always enabled
  - **Abuse.ch SSL blocklist** — enabled
  - **PT Research** — disabled by default (too noisy for SMB)
- Logging: alerts → local file + forwarded to Loki per `infra/0006-logging`
- Resource budget: 512 MB RAM, 1 vCPU core (sized at FW VM provisioning)
- Tunables: HOME_NET = `grp_net_internal` alias content; EXTERNAL_NET = `!HOME_NET`

### 2. IDS → IPS promotion gate

Promotion from alert-only to drop is gated by:

- Minimum **30 days** of alert-only operation
- **Documented false-positive rate** below 5% (count manually from Loki dashboard)
- **Explicit operator decision** recorded as a new ADR amendment or operating note
- Rollback plan: single-toggle IPS=off in playbook variable + `ansible-playbook --tags ids`

Never auto-promote based on time alone.

### 3. Internet-by-Request (IBR)

Pattern: a host normally in a default-deny outbound zone (Storage, CCTV, or SVC during maintenance) is **temporarily added to a dynamic alias** for a bounded time window. Used for OS updates, license activation, package fetches.

**Firewall plumbing (this ADR's scope):**

| Item | Value | Naming source |
|---|---|---|
| Alias | `host_internet_request` (type=host, empty at boot) | `naming/0003-firewall §1` |
| Rule | `PASS InternetRequest→WAN default` | `naming/0003-firewall §4` |
| NAT | Outbound NAT entry uses the alias as source net | (this ADR §4) |

**TTL automation (out of scope, contract only):**

- A separate service (web UI / CLI / bot) is the *only* component allowed to add or remove members of `host_internet_request`.
- Every membership change carries: target IP, owner, reason, TTL (max 4h for non-emergency, max 24h for emergency).
- The TTL service is responsible for removing entries on expiry — no FW change required.
- Every add/remove is logged to Loki (label `service=ibr`) and to RAID for audit.

This ADR does not specify the implementation of the TTL service. It only fixes the FW contract so different implementations (web UI per screenshot, Slack bot, GitLab pipeline) interoperate.

### 4. Audit + failure modes

- **Every IBR add/remove** → Loki + RAID log entry (`service=ibr`, `action=add|remove`, `target`, `owner`, `reason`, `ttl`)
- **Every Suricata alert** → Loki (`service=ids`, `signature_id`, `severity`, `src`, `dst`)
- **Suricata failure** (process crash, ruleset update fail) → does **not** affect packet forwarding (alert mode) and does **not** block FW management API. Health-check via OPNsense API.
- **IBR alias unreachable** (TTL service dead) → existing entries remain until manually removed; no silent expiry. The FW does not auto-purge.

### 5. Cross-references

- Aliases and rule naming **strictly** follow `naming/0003-firewall` — no `alias_` prefix, atomic where possible.
- Alert log shipping per `infra/0006-logging`.
- API credentials per `security/0001-secret-storage`.
- WAN segment definition per `infra/0004-network-architecture`.

## Consequences

- **Suricata IDS active from day one** on WAN1 — baseline visibility into edge attacks.
- **IPS promotion is a deliberate ADR-tracked decision**, not a silent toggle. Prevents accidental traffic drops on FP.
- **Internet-by-Request rule exists in the FW even before its TTL service is built** — implementers of the service can start with the FW already prepared (alias is empty, rule is enabled but matches nothing until the alias has entries).
- **Audit trail is non-optional** for IBR — operator cannot grant outbound without leaving a Loki + RAID footprint.
- **Suricata resource budget is fixed** at FW VM sizing time — overshooting requires a sizing review, not a quiet bump.

## Revision triggers

Revise when:
- A different IDS engine is adopted (Zeek, Snort 3) — replaces §1 and §2
- IPS-drop is promoted across the board — promotion conditions of §2 are recorded as fulfilled
- A second WAN (Telenet) enters production — §1 extends to WAN2
- IBR moves to a portal that supports OIDC / Authentik SSO — §3 contract extends with authentication identity
- The TTL service grows persistent state — separate ADR is created for that service

## CISO mapping

| Framework | Controls covered |
|---|---|
| ISO 27001:2022 | A.5.7 (threat intelligence — IDS feeds), A.8.16 (monitoring — alerts + Loki), A.8.20 (networks security — IDS at edge), A.8.32 (change management — IBR audit trail) |
| NIS2 | Art. 21(2)(b) (incident handling — IDS alert pipeline), Art. 21(2)(e) (network and information systems security) |
