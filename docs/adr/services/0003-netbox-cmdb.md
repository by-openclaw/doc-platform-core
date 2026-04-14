# services/0003 — NetBox CMDB

**Status:** Draft
**Date:** 2026-04-14 (supersedes flat ADR-0009, 2026-03-31)
**Scope:** How NetBox works as the platform CMDB — data model, role vocabulary, plugin pipeline, sync architecture, and deployment parameters. Does **not** re-assert "NetBox is the source of truth" — that is already stated in `naming/0001-infra §1`, `infra/0001-platform-stack`, and referenced from `infra/0007-monitoring`, `security/0001-secret-storage`, `services/0004-database-strategy`.
**Related:** `naming/0001-infra`, `infra/0001-platform-stack`, `infra/0003-terraform-standard`, `infra/0004-network-architecture`, `infra/0007-monitoring`, `services/0004-database-strategy`

---

## Context

Multiple merged ADRs assert "NetBox is the source of truth" as a rule: `naming/0001-infra §1`, `infra/0007-monitoring` (`prometheus_job` custom field), `security/0001-secret-storage` (service vocabulary match), etc. None of them define **how** NetBox actually implements that role — what data model, what custom objects, what role vocabulary, what sync mechanisms.

This ADR **completes** the model: it specifies the data, the roles, the plugin pipeline for missing domains, and the sync architecture between NetBox and the rest of the platform (Ansible, Terraform, Neo4j, Prometheus, Vault).

## Decision

### 1. Data model — what NetBox holds

NetBox holds **every managed asset on the platform** — physical, virtual, logical, and software. No asset exists outside NetBox. If it isn't in NetBox, it isn't managed.

**Physical assets:**
- Servers, switches, firewalls, NAS, WiFi access points
- PDUs, UPS, rack-mounted equipment, patch panels
- Cables, power feeds, circuits
- Secondary devices: printers, TVs, mobile management stations (when part of the platform scope)

**Virtual assets:**
- VMs (Proxmox + cloud: Contabo, Hetzner)
- LXC containers on Proxmox
- Kubernetes nodes (k3s) — when deployed

**Logical assets:**
- Interfaces, IP addresses (v4 + v6), prefixes, VLANs, VRFs
- Cables, patch-panel terminations
- DNS zones (as reference — authoritative source is the DNS server)

**Software / services:**
- Deployed platform tools modeled as NetBox `Service` objects (or custom objects in 4.x)
- Service endpoints (ports, protocols, TLS status)
- Application configuration metadata (admin URL, API endpoint, version)

**Custom objects (NetBox 4.x):**
- Domain-specific entities that don't fit the stock data model
- Examples: NAS shared folders, NFS export rules, group ACLs, quotas, OPNsense zones, WireGuard peer registrations

**Out of scope for NetBox:**
- Runtime metrics (Prometheus)
- Log streams (Loki)
- Secret values (HashiCorp Vault — NetBox may reference a Vault **path**, never a secret value)
- Audit logs (forwarded to Loki)
- User passwords (Authentik)

### 2. Mandatory custom fields

Every deployed asset carries the following NetBox custom fields — these are the **interface between NetBox and the rest of the platform ADRs**:

| Field | On | Type | Purpose | Consumed by |
|---|---|---|---|---|
| `env` | Device, VM, LXC | Choice (`dev`/`test`/`staging`/`acc`/`prod`/`drp`) | Environment tier | `infra/0005-environment-tiers`, Vault path, DNS zone |
| `role` | Device, VM, LXC | Choice (see §3) | Platform service role | `naming/0001-infra §5`, monitoring labels |
| `service_url` | VM, LXC | URL | User-facing HTTPS URL (e.g. `gitlab.by-research.be`) | DNS CNAME target, Traefik routing |
| `prometheus_job` | VM, LXC | Text | Prometheus `job=` label | `infra/0007-monitoring` — observability registry pattern |
| `vault_path_prefix` | VM, LXC | Text | `secret/{env}/{service}` — root of this asset's Vault scope | `security/0001-secret-storage` |
| `backup_class` | VM, LXC | Choice (see `infra/0008 §1`) | Service class → backup policy | `infra/0008-backup-strategy` |
| `owner` | Device, VM, LXC | Contact reference | Human or team responsible | RCA, incident response |

Adding a new mandatory custom field requires an ADR amendment (this ADR).

### 3. Canonical role vocabulary

The `role` custom field is a **closed choice list**. The list below is the initial canonical vocabulary — new roles are added via amendment to this ADR, not by operators adding values in the NetBox UI.

| Role | Example asset | Example name | Layer (`infra/0002`) |
|---|---|---|---|
| `hypervisor` | Proxmox VE host | `br-pmox-01` | Layer 1 |
| `firewall` | OPNsense | `vm-opns-01` | Layer 2 (prereq) |
| `storage` | Synology DSM, MinIO | `br-syno-01`, `vm-minio-01` | Layer 4 |
| `pki` | step-ca | `vm-stca-01` | Layer 2 |
| `secrets` | HashiCorp Vault | `vm-hvlt-01` | Layer 2 |
| `pwmgr` | Vaultwarden | `vm-vwrd-01` | Layer 5 |
| `identity` | Authentik | `vm-auth-01` | Layer 3 |
| `cmdb` | NetBox (self) | `vm-nbox-01` | Layer 5 |
| `dns` | Pi-hole, Unbound | `vm-phole-01`, `vm-unbnd-01` | Layer 5 |
| `proxy` | Traefik | `vm-trfk-01` | Layer 5 |
| `git` | GitLab CE | `vm-glab-01` | Layer 5 |
| `runner` | GitLab Runner | `vm-glrn-01` | Layer 5 |
| `db` | PostgreSQL | `vm-pgsql-01` | Layer 5 |
| `cache` | Redis | `vm-redis-01` | Layer 5 |
| `mon` | Prometheus, Grafana, Alertmanager | `vm-prom-01`, `vm-graf-01` | Layer 5 |
| `log` | Loki, Promtail | `vm-loki-01` | Layer 5 |
| `mail` | Mailcow | `vm-mcow-01` / `cb-mcow-01` | Layer 5 |
| `vpn` | WireGuard endpoint (on OPNsense) | — (interface on firewall) | Layer 2 |
| `app` | Generic application VM (fallback role) | `vm-nextcloud-01` | Layer 5 |

**Rules:**
- New roles are added by ADR amendment (edit this table, submit PR)
- The role token is used as the `service` segment in `naming/0001-infra §5` hostnames (with the mapping: `cmdb` → `nbox` short code, `identity` → `auth` short code, etc.)
- Roles that straddle multiple assets (e.g. `mon` covers Prometheus + Grafana + Alertmanager) use tags to distinguish the individual component

### 4. Plugin pipeline — when a plugin is missing

When NetBox's stock data model does not cover a domain (example: Synology NAS shared folders, OPNsense firewall alias registry), **develop a plugin**. Do not store the data in a YAML file, a wiki page, or "temporarily" in Ansible vars. NetBox is the only acceptable home.

| Phase | Scope | Deliverable |
|---|---|---|
| **Phase 1 — Minimum viable** | List, CRUD, read-back actual state from the target system | Working plugin, installable, CI green |
| **Phase 2 — Integration** | Relationships to other NetBox objects, webhook triggers, Ansible dynamic inventory support | Ansible reads NetBox for this domain via `netbox.netbox.nb_inventory` |
| **Phase 3 — Graph** | Neo4j sync nodes + edges, agent-queryable metadata | Agent (Rune) can reason about this domain via graph queries |

**Rules:**
- No phase is skipped. No Phase 3 without a working Phase 1.
- Plugin repo name: `netbox-plugin-{domain}` per `naming/0004-automation §1`
- Language: Python (NetBox plugin standard). Minimal runtime dependencies.
- DoD: tested, linted, CHANGELOG, CI gate — same as every other platform repo per `git/0001-workflow`
- Plugin development effort is **time-boxed per phase.** No gold-plating Phase 1.

### 5. Sync architecture

NetBox is the intent layer. Other systems read from NetBox; none of them are authoritative for any field NetBox models.

| Consumer | Read mechanism | Cadence |
|---|---|---|
| **Ansible** | `netbox.netbox.nb_inventory` dynamic inventory plugin | Every playbook run — live query |
| **Terraform** | `terraform-provider-netbox` data sources + resources | Read on every `plan`, write on `apply` for NetBox-managed resources |
| **Prometheus** | `netbox_sd` service discovery (future) or static config generated from NetBox | Scrape config regenerated on NetBox webhook |
| **Traefik** | Dynamic config generated from NetBox `service_url` custom field | On change via Ansible reconciliation |
| **Neo4j** | ETL job — reads NetBox API, writes nodes + edges | Event-driven via NetBox webhook + safety-net cron every 4h |
| **Agent (Rune)** | Queries Neo4j via Cypher for RCA, blast-radius, dependency traversal | Per user request |

**Write path — who updates NetBox:**

| Writer | What | When |
|---|---|---|
| **Operator (human)** | Intent: creates or edits an asset record before it is provisioned | Manual, via NetBox UI or API |
| **Ansible** | Status updates after provisioning: `status = active`, last-check timestamp, discovered interfaces | Post-playbook task in every role |
| **Terraform** | Resources it creates (when using `terraform-provider-netbox`): VMs, VLANs, IPs | On `terraform apply` |
| **NetBox plugins** | Discovered state from target systems: actual NFS shares, actual firewall aliases, actual WireGuard peers | On discovery cycle (per-plugin cadence) |

**Drift handling:** NetBox always holds intent. If a plugin or Ansible discovers drift (actual state ≠ NetBox intent), the drift is logged to Loki (see `infra/0006-logging §Audit log destinations`), flagged on the NetBox record with a `drift` tag, and either auto-reconciled (if safe) or queued for human review.

### 6. NetBox → Neo4j graph sync

NetBox's relational model cannot express dependency chains, impact analysis, or multi-hop traversal efficiently. Neo4j is used as a **graph projection** of NetBox's data, updated on every NetBox change.

**Node types (initial):**
`Device`, `VM`, `LXC`, `Service`, `Share`, `Group`, `User`, `VLAN`, `Interface`, `IPAddress`, `Secret` (reference only, no value), `DNSRecord`

**Edge types (initial):**
`runs_on`, `mounts`, `has_access`, `member_of`, `connected_to`, `depends_on`, `exposes`, `manages`, `routes_to`, `belongs_to_env`

**Example query (RCA):**
```
MATCH (v:VM {name: 'vm-glab-01'})-[:depends_on*1..3]->(dep)
WHERE dep.status <> 'healthy'
RETURN dep, collect(nodes(p))
```

**Sync implementation:** custom Python ETL job, deployed as a NetBox plugin that uses NetBox webhooks + a 4-hour safety-net cron. Neo4j is **read-only** for the platform — no application writes to Neo4j directly, every change is ETL-ed from NetBox.

### 7. NetBox deployment parameters

NetBox itself is a Layer 5 service per `infra/0002-platform-charter`. Its deployment follows the generic Linux VM pipeline (not a provisioning exception like `services/0001-opnsense`).

| Parameter | Value |
|---|---|
| **Hostname** | `vm-nbox-01` per `naming/0001-infra §5` |
| **Service URL** | `netbox.{domain}` (prod), `netbox.{env}.{domain}` (non-prod) per `naming/0001-infra §7` |
| **Version** | NetBox **4.x** minimum (custom objects require 4.x) |
| **Database** | PostgreSQL — consumer of the shared instance per `services/0004-database-strategy §Consumers` |
| **Cache** | Redis — consumer of the shared instance per `services/0004-database-strategy §Consumers` |
| **Object storage** | MinIO for image uploads, exports, attachments |
| **Authentication** | Authentik OIDC per `identity/0001-authentication`. Break-glass local admin account in Vault at `secret/{env}/netbox/break-glass`. |
| **Secrets** | API tokens, DB credential, OIDC client secret all in Vault at `secret/{env}/netbox/...` per `security/0001-secret-storage` |
| **Role in own registry** | `role = cmdb` — NetBox is modeled as an asset in itself |

### 8. Deferred decisions

The following are intentionally **not** decided in this ADR; they require empirical data at deployment time.

| Decision | Deferred until |
|---|---|
| Priority order for plugin development (NAS vs OPNsense vs Ansible inventory vs other) | NetBox deployment + first gap encountered |
| Use NetBox 4.x custom objects vs NetBox plugin API for a given domain | Per-domain, evaluated at plugin kickoff |
| Neo4j ETL: custom script vs existing OSS connector (e.g. `netbox-graph`) | Neo4j deployment |
| Authentik → NetBox group sync: webhook push vs polling | Authentik identity-sync framework maturity |
| Which tools are modeled as NetBox `VirtualMachine` vs `Service` vs custom object | Per-tool, at deployment time |

These deferred decisions are **not TBD** in the `security/0003-hardening` sense — they are intentionally empirical and will be recorded as per-plugin or per-deployment notes, not as ADR amendments.

## Consequences

- **Single source of truth is real, not aspirational.** NetBox holds every field; Ansible and Terraform are executors that read from NetBox and write back status.
- **Role vocabulary is closed and versioned.** Operators cannot add ad-hoc role strings — every new role requires an ADR amendment, which keeps the vocabulary consistent across monitoring, naming, and backup.
- **Mandatory custom fields are the contract between NetBox and other ADRs.** `env`, `role`, `service_url`, `prometheus_job`, `vault_path_prefix`, `backup_class`, `owner` — if any of these is missing on a deployed asset, the deployment is incomplete.
- **Missing plugins become in-house plugins.** No YAML-file sources of truth, no temporary wiki pages. The 3-phase pipeline is mandatory.
- **Graph queries are possible** (RCA, blast radius, dependency traversal) via Neo4j — but only for data that has been modeled in NetBox first.
- **NetBox is itself in NetBox** (`role = cmdb`) — no special case.

## Revision triggers

Revise this ADR when:
- A new role is needed (add a row to §3)
- A new mandatory custom field is required (add to §2)
- NetBox replaces its plugin API (would affect §4)
- Neo4j is replaced by a different graph store (or graph sync is abandoned)
- A consumer is added to §5 (e.g. OPA / Kyverno policy engine reading NetBox)
- The ETL mechanism changes (custom script → OSS connector → native NetBox 5.x feature)
- NetBox 5.x data-model changes force a structural update

## CISO mapping

| Framework | Controls covered |
|---|---|
| ISO 27001:2022 | A.5.9 (inventory of information and other assets — NetBox holds every asset), A.8.9 (configuration management — mandatory custom fields are the contract), A.5.12 (classification of information — role vocabulary classifies every asset by function) |
| NIS2 | Art. 21(2)(a) (risk management — asset visibility via NetBox is a prerequisite for risk assessment), Art. 21(2)(f) (cryptography use policies — NetBox models the topology that cryptographic controls protect) |
