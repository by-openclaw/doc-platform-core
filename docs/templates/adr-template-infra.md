<!--
  SCOPE GUARD — INFRA ADR
  ========================
  This template is for infrastructure decisions only.
  DO NOT use this template for:
  - Library implementation details (DI, test strategy, log format)
  - Developer tooling (devcontainer, pre-commit hooks)
  - App-level credential handling
  Wrong template = PR blocked.
-->

# ADR-XXXX: Title in sentence case

<!-- One line. Sentence case. No trailing period. Example: "Network VLAN architecture for PoC platform" -->

**Status:** Draft | Accepted | Deprecated | Superseded by ADR-XXXX
**Date:** YYYY-MM-DD
**Deciders:** @yboujraf

---

## Context

<!-- 
  What problem is this solving? Which environment tier(s) are affected?
  - State the situation before the decision was made.
  - Reference relevant prior ADRs if applicable.
  - Do NOT describe the decision here — that goes in the next section.
  Keep to 2–5 sentences or a short list.
-->

## Decision

<!-- 
  One clear statement of what was decided.
  Start with a bold summary sentence, then detail below.
  Example: "**Deploy Vault on SVC VLAN using step-ca TLS with internal root CA.**"
-->

## VM / Resource Spec

<!-- 
  OMIT THIS SECTION if the decision does not involve VM provisioning.
  Fill in all rows; write "N/A" if a field does not apply.
-->

| Field        | Value                     |
|--------------|---------------------------|
| VM name      | <!-- e.g. svc-vault-01 --> |
| vCPU         |                           |
| RAM          |                           |
| Disk         |                           |
| Storage pool | <!-- e.g. local-zfs -->   |
| VLAN         | <!-- VLAN ID + name -->   |
| IP           |                           |
| OS           |                           |
| Template     |                           |

## Network

<!--
  Document all network exposure for this decision.
  - Ports: list each port, protocol, and direction (inbound/outbound)
  - Zones: source and destination VLAN(s), referencing ADR-0015 VLAN registry
  - Firewall rules: use named aliases per ADR-0015 OPNsense standard (no hardcoded IPs)
  - Traefik config: entrypoint, router rule, middleware chain — if this service is publicly exposed
  If not network-relevant, write "Not applicable" and remove the table.
-->

| Port | Protocol | Direction | Source zone | Destination zone | Purpose |
|------|----------|-----------|-------------|------------------|---------|
|      |          |           |             |                  |         |

**Firewall aliases required:**

| Alias | Type | Value |
|-------|------|-------|
|       |      |       |

## Storage

<!--
  Specify all storage backends used by this decision.
  - Backend: MinIO / Contabo S3 / NAS (Synology) / local / Proxmox Ceph
  - Volume names, mount paths
  - Backup policy: what is backed up, how often, retention, target
  If not storage-relevant, write "Not applicable."
-->

| Component   | Backend | Volume / Path | Backup policy |
|-------------|---------|---------------|---------------|
|             |         |               |               |

## TLS / PKI

<!--
  Every service that has a TLS endpoint must document its certificate model.
  - CA: step-ca (internal) OR Let's Encrypt (public) — choose one, state why
  - Trust model: how does the client trust the cert? (root CA distributed to clients, public CA, etc.)
  - No-warning guarantee: confirm the cert chain produces zero browser/client warnings
  - Renewal: automatic (step-ca ACME / certbot) or manual? If manual, how is rotation tracked?
  If this ADR has no TLS component, write "Not applicable."
-->

| Field              | Value                            |
|--------------------|----------------------------------|
| CA                 | <!-- step-ca / Let's Encrypt --> |
| Cert type          |                                  |
| Trust model        |                                  |
| No-warning guarantee | <!-- Yes / No — explain if No --> |
| Renewal method     |                                  |
| Cert expiry        |                                  |

## CISO mapping

> Applies only to controls directly relevant to this ADR's scope.
> Do NOT list every ISO control — only those this ADR satisfies, partially satisfies, or gaps.

### ISO/IEC 27001:2022

| Control | Title | Status | Notes |
|---|---|---|---|
| A.X.XX | Control title | ✓ Covered / ⚠ Partial / ✗ Gap | One-line explanation |

### NIS2 (Directive 2022/2555)

| Article | Requirement | Status | Notes |
|---|---|---|---|
| Art. XX(X) | Requirement text | ✓ / ⚠ / ✗ | One-line explanation |

### GDPR (Regulation 2016/679)

| Article | Requirement | Status | Notes |
|---|---|---|---|
| Art. XX | Requirement text | ✓ / ⚠ / ✗ | One-line explanation |

## Licensing

<!--
  For every tool or service introduced by this decision:
  - SPDX license identifier (https://spdx.org/licenses/)
  - Free vs paid tier — specify if paid: cost model, who approves spend
  - Link to the license file in the relevant repo or vendor docs
  If no new tools are introduced, write "No new tools introduced."
-->

| Tool / Service | SPDX license | Tier      | License ref |
|----------------|--------------|-----------|-------------|
|                |              |           |             |

## Consequences

<!--
  What does this decision enable? What does it constrain?
  List known risks — be honest. If there's a known gap or future work required, say so.
  Structure: positive consequences first, then constraints, then risks.
-->

**Enables:**
-

**Constrains:**
-

**Known risks:**
-

## References

<!--
  External links: vendor docs, RFCs, security advisories, runbooks.
  Internal: link to relevant docs/stack.md entries, runbooks, or tooling configs.
-->

-
