# security/0005 — Open Source Licensing Policy

**Status:** Accepted
**Date:** 2026-04-14 (supersedes flat ADR-0022, 2026-04-02) · **flagged-license decisions signed off 2026-09-09 by @yboujraf** (§4)
**Scope:** Licensing rules for all tools and libraries deployed on the platform — approved SPDX license list, flagged licenses requiring assessment, internal-use-only model, BoM tracking requirements. Does **not** define tool selection or platform stack inventory.
**Related:** `security/0002-compliance-mapping`, `infra/0001-platform-stack`, `lib/python/0001-design-standard`

---

## Context

BY-SYSTEMS is a **systems integrator** operating its own internal platform infrastructure and hosting services for its own operational purposes. It does **not** redistribute software as SaaS, does **not** resell open-source tools to third parties, and does **not** offer software as a service to external customers.

The licensing question is therefore:
- **Can we legally deploy and operate this tool internally?**

Not:
- "Can we redistribute it?"
- "Does this obligate us to publish our own source?"
- "Does hosting this tool for our customers trigger the SaaS clauses?"

Despite this narrower scope, licensing still matters because:

- **Redis** relicensed to **SSPL-1.0** from v7.4+ — SSPL's definition of "Service" is broad; internal-use assessment required before upgrade
- **HashiCorp Vault** uses **BUSL-1.1** — "production use" restriction until 4-year conversion to MPL; internal production deployment requires review
- Without a platform-wide license tracking policy, tools are adopted ad-hoc without a license record on file — incompatible with ISO 27001:2022 A.5.20 and supply-chain audit requirements

## Decision

### 1. Internal-use-only assessment model

**All licensing assessments on this platform apply to internal deployment and operation only.** Redistribution and SaaS clauses in any license (SSPL, BUSL, AGPL, commercial) are not triggered because BY-SYSTEMS does not redistribute or offer SaaS.

When a license is assessed, the question is:
- Does this license permit us to install, configure, operate, and expose this tool to our own internal team on our own infrastructure?

If **yes** → the tool may be deployed.
If **no** → the tool is blocked.
If **uncertain** → written approval from @yboujraf before deployment or upgrade (see §3 Flagged licenses).

### 2. Approved SPDX licenses — no further review required

The following SPDX license identifiers are **approved for internal deployment** without additional review:

| Category | SPDX IDs |
|---|---|
| Permissive | `MIT`, `Apache-2.0`, `BSD-2-Clause`, `BSD-3-Clause`, `ISC`, `Zlib` |
| Weak copyleft | `LGPL-2.1`, `LGPL-2.1-only`, `LGPL-2.1-or-later`, `LGPL-3.0`, `LGPL-3.0-only`, `LGPL-3.0-or-later`, `MPL-2.0` |
| Strong copyleft (non-SaaS) | `GPL-2.0`, `GPL-2.0-only`, `GPL-2.0-or-later`, `GPL-3.0`, `GPL-3.0-only`, `GPL-3.0-or-later` |

**Rules:**
- Any tool whose license is in this list can be deployed without written approval — the license record (SPDX ID + source URL) still must be captured in the tool's `docs/licensing.md` file
- `GPL-*` is approved because BY-SYSTEMS does not distribute the tool to third parties; the copyleft obligation triggers on distribution, not on internal use

### 3. Flagged licenses — written approval required before adoption

The following SPDX identifiers require **written approval from @yboujraf** before the tool is adopted, upgraded, or touched at all:

| SPDX ID | License | Why flagged |
|---|---|---|
| `BUSL-1.1` | Business Source License 1.1 | "Production use" restriction until conversion date; internal-use assessment required |
| `SSPL-1.0` | Server Side Public License 1.0 | Broad "Service" definition; internal-use assessment required |
| `AGPL-3.0`, `AGPL-3.0-only`, `AGPL-3.0-or-later` | GNU Affero GPL v3 | "Remote network interaction" copyleft trigger. **Covered by the blanket assessment in §4 while the tool runs unmodified** — a patched AGPL tool served over the network needs its own assessment issue |
| `Commons Clause` | Commons Clause addendum | Commercial-use restriction; internal-use assessment required |
| `Elastic-2.0`, `ELv2` | Elastic License v2 | Restricts hosted SaaS, managed service, "circumvention" clauses |
| `Proprietary`, `Commercial`, any non-OSI license | — | Not open source; separate approval process applies |

**Assessment process:**
1. Open an issue titled `license-assessment: {tool}` in `doc-platform-core`
2. Attach the license text, the tool's `README` / `LICENSE` file content, and the use case
3. @yboujraf writes a short assessment in the issue: "approved for internal use because …" or "blocked because …"
4. On approval: the tool is added to `docs/licensing.md` with SPDX ID, assessment issue link, and date
5. Assessment decisions are tracked in `security/0002-compliance-mapping` as inputs to ciso-assistant

**Self-assessment is sufficient** for internal-use decisions by @yboujraf — no external legal review required unless a tool is being evaluated for customer-facing deployment (out of scope for this ADR).

### 4. Flagged license decisions (current)

The following flagged licenses have been **assessed and approved for internal deployment** on BY-SYSTEMS infrastructure. Per-tool decisions first, then the two class-wide ones (AGPL blanket, Terraform):

| Tool | SPDX ID | Status | Assessment |
|---|---|---|---|
| **Redis** (≥ 7.4) | `SSPL-1.0` (dual-licensed with `RSAL-v2`) | ✅ Approved | SSPL permits internal deployment and operation; no redistribution or SaaS offering is involved. License terms respected as-is. |
| **HashiCorp Vault** | `BUSL-1.1` | ✅ Approved | Internal production deployment is within BUSL-1.1 terms for an integrator operating its own infrastructure. Converts to `MPL-2.0` 4 years after each release. License terms respected as-is. |


#### AGPL-3.0 — blanket assessment for unmodified network services

**Decision (2026-09-09, @yboujraf):** `AGPL-3.0` (and `-only` / `-or-later`) is **approved for internal
deployment without a per-tool assessment issue**, on one condition: the tool is run **unmodified**.

Reasoning: AGPL §13 adds one obligation on top of GPL-3.0 — if you *modify* the program and let users
interact with it *over a network*, you must offer those users the modified source. We deploy upstream
releases (distro packages, upstream container images, upstream tarballs) with configuration only.
Configuration is not modification of the program, so §13 never triggers. Nothing is redistributed and
nothing is offered as SaaS (§1).

**The condition is the control:** the moment we patch an AGPL tool's source and run the result as a
network service, that tool leaves this blanket and needs its own assessment issue plus a plan to publish
the modified source. Building an image *from* an unmodified upstream release (pinning, adding config,
adding a wrapper entrypoint) stays inside the blanket; patching the application does not.

AGPL tools currently deployed under this blanket — all unmodified upstream releases:

| Tool | Role | Owning repo / role |
|---|---|---|
| Proxmox VE / Proxmox Backup Server | hypervisor, backups | `infra-terraform-proxmox`, `ansible-platform/roles/pbs` |
| Nextcloud | files, contacts | `ansible-platform/roles/nextcloud` |
| Grafana | dashboards | `ansible-platform/roles/grafana` |
| Loki + Promtail | log aggregation, shipping | `ansible-platform/roles/loki`, `roles/promtail` |
| Vaultwarden | password manager | `ansible-platform/roles/vaultwarden` |
| Monit | firewall process alerting (`os-monit`) | `ansible-platform/roles/opnsense` (catalog `opnsense_monit_*`) |
| Redis (v8, AGPL-3.0 option of its tri-license) | cache | `ansible-platform/roles/redis` |

Redis 8 is tri-licensed (`RSAL-v2` / `SSPL-1.0` / `AGPL-3.0`); we take the **AGPL-3.0** option, which is
the OSI-approved one, so it falls under this blanket rather than under the SSPL row above.

#### Terraform

| Tool | SPDX ID | Status | Assessment |
|---|---|---|---|
| **HashiCorp Terraform** | `BUSL-1.1` | ✅ Approved (2026-09-09) | Same reasoning as Vault: internal production use by an integrator operating its own infrastructure is within BUSL-1.1; converts to `MPL-2.0` four years after each release. Migration target **OpenTofu** is on file below and already named in `infra/0003-terraform-standard`. |

**Alternatives kept on file** in case of future license tightening:

| Flagged tool | OSS alternative | Notes |
|---|---|---|
| Redis | **Valkey** (BSD-3-Clause) or **KeyDB** (BSD-3-Clause) | Linux Foundation community forks, drop-in API compatible |
| HashiCorp Vault | **OpenBao** (MPL-2.0) | Linux Foundation fork of Vault; feature parity maintained |
| HashiCorp Terraform | **OpenTofu** (MPL-2.0) | Linux Foundation fork of Terraform; already referenced in `infra/0003-terraform-standard §Revision triggers` |

If any of the flagged licenses tighten in a future version (e.g. SSPL scope expands, BUSL converts to a stricter license before the MPL conversion date), the tool is migrated to the OSS alternative.

### 5. Permission matrix — what we may and may not do

Per license class, for **BY-SYSTEMS as an internal operator** (§1). "Modify" means changing the
program's own source, not writing configuration for it.

| We want to … | Permissive (MIT, Apache-2.0, BSD, ISC) | Weak copyleft (LGPL, MPL-2.0) | Strong copyleft (GPL-2.0/3.0) | Network copyleft (AGPL-3.0) | Source-available (BUSL-1.1, SSPL-1.0, ELv2) |
|---|---|---|---|---|---|
| Install and run it on our own infrastructure | ✅ | ✅ | ✅ | ✅ | ✅ (assessed, §4) |
| Expose it to our own staff over the network | ✅ | ✅ | ✅ | ✅ | ✅ |
| Write configuration, roles, catalogs for it | ✅ | ✅ | ✅ | ✅ | ✅ |
| Keep our own configuration and glue code private | ✅ | ✅ | ✅ | ✅ | ✅ |
| Modify the program's source for internal use only | ✅ | ✅ (publish changes to the library on distribution) | ✅ | ⚠️ must offer the modified source to network users → leaves the blanket, needs its own assessment | ❌ without a per-case assessment |
| Redistribute binaries or images to a third party | ✅ (keep notices) | ⚠️ conditions apply | ⚠️ must ship source | ⚠️ must ship source | ❌ |
| Host it **for a customer** as a service | ✅ | ✅ | ✅ | ⚠️ source-offer duty if modified | ❌ — out of scope for this ADR, needs legal review |
| Resell it, or build a competing hosted product on it | ✅ | ✅ | ✅ | ⚠️ | ❌ (the exact clause these licenses exist for) |

The last two rows are the boundary of this ADR: BY-SYSTEMS is an integrator running its own platform.
**Any customer-facing or resale scenario voids every ✅ in this table and requires a fresh review**
(§7), because that is precisely the case BUSL / SSPL / ELv2 restrict and the case where AGPL's network
clause starts to matter to someone other than us.

### 6. Ownership and where the license record lives

Two different documents, deliberately not merged:

| File | Covers | Content |
|---|---|---|
| `LICENSE` (repo root) | **our own code** | MIT for every BY-SYSTEMS repo (`ansible-platform`, `lib-opnsense`, `ansible-opnsense`, `infra-terraform-proxmox`, `doc-platform-core`) |
| `docs/licensing.md` (per repo) | **everything we deploy or depend on** | the third-party BoM: SPDX ID, class (Approved / Flagged), version pin, owning role, assessment reference — fields in §7 |

`docs/licensing.md` states in its first paragraph that our own code is MIT under `LICENSE`, so neither
file has to be read to understand the other.

**Owner:** @yboujraf is the accountable owner for every license decision on the platform. The
*technical* owner of a component — who bumps its pin and therefore who must update its BoM row in the
same PR — is the repo and role that deploys it:

| Domain | Technical owner (repo / role) | Components |
|---|---|---|
| Edge & routing | `ansible-platform/roles/opnsense` + `infra-terraform-proxmox/modules/vm-opnsense` | OPNsense, Suricata, Unbound, dnscrypt-proxy, Kea, radvd, chrony, Monit, lldpd, acme.sh, ddclient, mdns-repeater, syslog-ng |
| Virtualisation & backup | `infra-terraform-proxmox`, `ansible-platform/roles/pbs` | Proxmox VE, Proxmox Backup Server |
| Identity & secrets | `ansible-platform/roles/{authentik,vault,vaultwarden,stepca}` | Authentik, Vault, Vaultwarden, step-ca |
| Edge TLS & exposure | `ansible-platform/roles/{traefik,netbird,adguard}` | Traefik, NetBird, AdGuard Home |
| Detection & response | `ansible-platform/roles/{crowdsec,crowdsec_agent,crowdsec_fw_bouncer}` | CrowdSec LAPI, agents, bouncers |
| Observability | `ansible-platform/roles/{loki,promtail,grafana,prometheus}` | Loki, Promtail, Grafana, Prometheus |
| Data & storage | `ansible-platform/roles/{postgres,redis,seaweedfs,pgadmin}` | PostgreSQL, Redis, SeaweedFS, pgAdmin |
| Developer platform | `ansible-platform/roles/{gitlab,harbor,verdaccio,jumpserver}` | GitLab CE, Harbor, Verdaccio, JumpServer CE |
| Collaboration | `ansible-platform/roles/{mailcow,nextcloud,netbox}` | Mailcow, Nextcloud, NetBox |
| Automation itself | `ansible-platform`, `lib-opnsense`, `ansible-opnsense` | ansible-core + collections, our own MIT libraries |

**Dependencies:** a BoM row records a component's own runtime dependencies only where the dependency
carries a *different or stricter* license class than the component (for example Redis under a service
that is otherwise Apache-2.0, or a GPL command-line tool invoked by an MIT role). Transitive library
graphs are not enumerated by hand — they are covered by the repo's dependency pins and, for container
images, by the upstream image's own manifest.

### 7. Bill of Materials (BoM) tracking

Every tool deployed on the platform has a mandatory `docs/licensing.md` file in its repository (or in `platform-setup/tools/{tool}/docs/licensing.md` for tools that don't have a dedicated repo). The file contains:

| Field | Required | Notes |
|---|---|---|
| `SPDX License` | ✅ | Use the official SPDX identifier — never "open source" or free text |
| `License URL` | ✅ | Direct link to the LICENSE file in the tool's upstream repo |
| `Source URL` | ✅ | Upstream project URL (GitHub, GitLab, etc.) |
| `Version constraint` | ✅ | What version or version range is approved (e.g. "Redis < 7.4" vs "Redis ≥ 7.4 SSPL") |
| `Assessment issue` | only for flagged licenses | Link to the GitHub issue where the flagged-license assessment was made |
| `Owner` | ✅ | The repo / role that deploys it (§6) — whoever bumps the pin updates this row |
| `Last reviewed` | ✅ | Date of last license review — reviewed annually at minimum |
| `BoM ref` | ✅ | Reference to the platform BoM entry where this tool appears |

**Platform BoM:** the full tool inventory with license columns is maintained as part of `infra/0001-platform-stack §Cross-reference`. When a tool is added or removed from the platform, both its `docs/licensing.md` and the platform BoM entry are updated in the same PR.

### 8. Review cadence

- **Per-tool:** every tool's `docs/licensing.md` is reviewed **annually** — verified that the SPDX ID is still current, the assessment (if flagged) is still valid, and no upstream license change has occurred
- **Platform-wide:** the full BoM is audited **annually** — every tool listed has an up-to-date `docs/licensing.md`, every flagged license has a current assessment
- **Event-driven:** when a vendor announces a license change (Redis → SSPL, Vault → BUSL, etc.), affected tools are re-assessed **immediately** and the outcome recorded

### 9. What this ADR does NOT cover

- **Customer-facing deployments** — if a tool is ever deployed for or resold to a customer, a separate legal review is required, outside the scope of this ADR
- **Proprietary / commercial software procurement** — this ADR covers OSS licensing only; commercial licenses have their own procurement process
- **Copyright ownership of BY-SYSTEMS code** — this ADR governs third-party tools only; BY-SYSTEMS's own code licensing policy is a separate concern
- **Export control / ECCN** — licensing is separate from export classification

## Consequences

- **Every tool has a documented SPDX license** — no silent deployments, no "we think it's open source"
- **Flagged licenses require explicit assessment** — BUSL / SSPL / AGPL / ELv2 / Commons Clause never slip in unnoticed
- **Internal-use model simplifies BUSL / SSPL / AGPL decisions** — we're an integrator, not a SaaS vendor, so redistribution and network-copyleft clauses don't trigger
- **OSS alternatives are documented** for every flagged license — if upstream tightens, the migration target is already known (Valkey for Redis, OpenBao for Vault, OpenTofu for Terraform)
- **BoM audit trail** — annual review + event-driven re-assessment + ciso-assistant integration gives ISO 27001 A.5.20 and NIS2 Art. 21(2)(d) evidence
- **Review cadence is defined** — no tool has an "expired" license assessment on file
- **AGPL no longer blocks routine work** — the blanket in §4 covers unmodified upstream releases, and the
  condition (do not patch and serve) is the tripwire that sends a tool back to a per-tool assessment
- **The permission matrix answers the day-to-day question** — "may we do X with this tool" is a table
  lookup, and the two rows that are red (customer hosting, resale) are the two that need a human
- **Our code and their code are separated** — `LICENSE` is ours (MIT), `docs/licensing.md` is theirs

## Revision triggers

Revise this ADR when:
- A new SPDX license category appears on the platform and doesn't fit existing approved / flagged lists
- A flagged license changes (upstream relicense, version bump with new terms, conversion date reached)
- BY-SYSTEMS's business model changes from systems integrator to SaaS vendor or software distributor — triggers re-assessment of every license on file
- A new compliance framework (DORA, SOC 2, PCI-DSS) requires different license tracking evidence
- An OSS alternative listed in §4 is itself deprecated or diverges enough that it's no longer a viable migration target

## CISO mapping

| Framework | Controls covered |
|---|---|
| ISO 27001:2022 | A.5.20 (addressing information security within supplier agreements — license policy defined for every supplier), A.8.30 (outsourced development — source available for all deployed OSS tools, license tracked per tool), A.5.23 (use of cloud services — all tools self-hosted, cloud-licensing clauses not triggered) |
| NIS2 | Art. 21(2)(d) (supply chain security — license review required per tool before deployment, BoM maintained) |
