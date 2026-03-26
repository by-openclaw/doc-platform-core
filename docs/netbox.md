# BY-SYSTEMS Platform — NetBox (Source of Truth)
**Last updated:** 2026-03-26
**Status:** Active
**Scope:** IPAM / DCIM / CMDB
**Related docs:** `docs/stack.md`, `docs/naming-convention.md`, `docs/idempotency-strategy.md`

> **NetBox is the single source of truth** for all infrastructure objects: devices, VMs, IP addresses, VLANs, VRFs, circuits, sites, racks, and services.
> All automation (Ansible, Terraform, Grafana, Prometheus) derives state from NetBox — not from local config files.

---

## 1. What NetBox Does

NetBox serves three core roles in the BY-SYSTEMS platform:

| Role | Scope | Examples |
|---|---|---|
| **IPAM** (IP Address Management) | IPv4/IPv6 prefixes, addresses, VLANs, VRFs | `10.0.10.0/24` (VLAN 10, management), `fd00::/48` (IPv6 ULA) |
| **DCIM** (Data Center Infrastructure Management) | Sites, racks, devices, power, cabling | `sw-arista-prod-01` in Rack A1, Site Brussels |
| **CMDB** (Configuration Management Database) | Services, custom fields, relationships | `gitlab` service on `vm-gitlab-prod-01`, version `17.0.1`, lifecycle `active` |

### What NetBox is NOT

- **Not a monitoring tool** — Prometheus/Grafana handle metrics and alerts
- **Not a secrets store** — Vault handles secrets
- **Not a graph database** — Neo4J handles dependency/relationship mapping (synced from NetBox)
- **Not a config management tool** — Ansible handles configuration; NetBox provides the inventory

---

## 2. Community Edition — Features and Limitations

NetBox is open source (Apache 2.0). BY-SYSTEMS uses the **community edition** (self-hosted).

### Included in Community (free)

| Feature | Notes |
|---|---|
| Full IPAM | Prefixes, IP addresses, VLANs, VRFs, aggregates, ASNs |
| Full DCIM | Sites, racks, devices, interfaces, cables, power |
| Services registry | Map services to devices/VMs with ports |
| Custom fields | Add arbitrary fields to any object type |
| Custom links | Add clickable links to object views |
| Tags | Free-form labels on any object |
| REST API | Full CRUD, filtering, pagination, bulk operations |
| GraphQL API | Read-only query interface |
| Webhooks | HTTP callbacks on object changes |
| Custom scripts | Python scripts executed within NetBox |
| Export templates | Jinja2 templates for custom data export |
| Journal entries | Per-object change notes |
| Config contexts | Hierarchical JSON config data attached to devices/VMs |
| NAPALM integration | Live device data (interfaces, LLDP, BGP) |
| Ansible inventory plugin | `netbox.netbox` collection — dynamic inventory from NetBox |
| Multi-tenancy | Tenants and tenant groups for customer isolation |
| Change logging | Full audit trail of all object changes |

### NOT in Community (requires NetBox Cloud or plugins)

| Feature | Workaround |
|---|---|
| SSO/OIDC built-in | Use `django-auth-oidc` plugin or Authentik proxy auth |
| Advanced RBAC | Basic permissions available; use API tokens per integration |
| Commercial support | Community support only (GitHub issues, Slack) |
| SLA guarantees | Self-hosted — availability is your responsibility |
| Managed backups | PostgreSQL `pg_dump` + MinIO (manual or Ansible-scheduled) |

---

## 3. NetBox Object Model (BY-SYSTEMS)

### Sites

| Object | Naming pattern | Example |
|---|---|---|
| Site | `{city-slug}` | `brussels`, `contabo-eu` |
| Region | `{country}` | `belgium`, `germany` |

### Racks

| Object | Naming pattern | Example |
|---|---|---|
| Rack | `{site}-rack-{number:02d}` | `brussels-rack-01` |

### Devices

Per `docs/naming-convention.md` §7:

```
{function}-{vendor}-{env}-{number:02d}
```

| Function | Vendor | Env | Example |
|---|---|---|---|
| `sw` (switch) | `arista` | `prod` | `sw-arista-prod-01` |
| `ap` (access point) | `unifi` | `prod` | `ap-unifi-prod-01` |
| `srv` (server) | `proxmox` | `prod` | `srv-proxmox-prod-01` |
| `fw` (firewall) | `pfsense` | `prod` | `fw-pfsense-prod-01` |

### Virtual Machines

```
vm-{service}-{env}-{number:02d}
```

Examples: `vm-gitlab-prod-01`, `vm-vault-prod-01`, `vm-netbox-prod-01`

### VLANs

| VLAN ID | Name | Purpose |
|---|---|---|
| 10 | `mgmt` | Management network |
| 20 | `platform` | Platform services |
| 30 | `storage` | Storage traffic (MinIO, NFS) |
| 40 | `monitoring` | Prometheus, Grafana, Loki, Zabbix |
| 50 | `dmz` | Public-facing services |
| 100 | `media-red` | Broadcast primary (VRF RED) |
| 101 | `media-blue` | Broadcast redundant (VRF BLUE) |
| 200 | `voip` | SIP/RTP traffic |
| 300 | `iot` | IoT devices, cameras |

### IP Prefixes

| Prefix | VLAN | Role | Status |
|---|---|---|---|
| `10.0.10.0/24` | 10 (mgmt) | Management | Active |
| `10.0.20.0/24` | 20 (platform) | Platform services | Active |
| `10.0.30.0/24` | 30 (storage) | Storage | Active |
| `fd00:10::/64` | 10 (mgmt) | IPv6 management | Active |

> Actual prefixes are examples — real values are maintained in NetBox, not in this document.

### VRFs

| VRF | Purpose | Enforce unique | Route distinguisher |
|---|---|---|---|
| `default` | General platform traffic | Yes | — |
| `RED` | Primary media (broadcast) | Yes | `65000:100` |
| `BLUE` | Redundant media (SMPTE 2022-7) | Yes | `65000:101` |
| `MGMT` | Management with leak to RED/BLUE | Yes | `65000:10` |

### Tenants

| Tenant slug | Name | Purpose |
|---|---|---|
| `by-systems` | BY-SYSTEMS | Internal platform |
| `{customer-slug}` | Customer name | Per-customer isolation |

> Tenant slug = customer slug used in GitLab subgroups, repo names, and K8S namespaces.

### Custom Fields (per service)

Every service registered in NetBox carries these custom fields:

| Field | Type | Example | Purpose |
|---|---|---|---|
| `repo_url` | URL | `https://gitlab.by-systems.internal/platform/gitlab-core` | Links to source repo |
| `prometheus_job` | Text | `gitlab` | Prometheus scrape job name |
| `grafana_tag` | Text | `service:gitlab` | Grafana dashboard tag |
| `version` | Text | `17.0.1` | Currently deployed version |
| `lifecycle` | Selection | `active` / `deprecated` / `eol` | Service lifecycle state |

### Tags

Tags provide cross-cutting classification:

| Tag | Purpose |
|---|---|
| `tier-1` | Base platform service |
| `tier-2-broadcast` | Broadcast module |
| `tier-2-voip` | VoIP module |
| `tier-2-cctv` | CCTV module |
| `ansible-managed` | Managed by Ansible automation |
| `terraform-managed` | Provisioned by Terraform |
| `ha-required` | Requires high availability in production |
| `backup-daily` | Daily backup via Borgmatic/pg_dump |

---

## 4. Platform Integrations

### 4.1 NetBox → Ansible (Dynamic Inventory)

NetBox replaces static inventory files. The `netbox.netbox` Ansible collection provides a dynamic inventory plugin.

```yaml
# inventory/netbox.yml
plugin: netbox.netbox.nb_inventory
api_endpoint: https://netbox.by-systems.internal
token: "{{ lookup('hashi_vault', 'secret/netbox/ansible-token:token') }}"
validate_certs: true
group_by:
  - site
  - device_role
  - tenant
  - tags
compose:
  ansible_host: primary_ip4.address | ansible.utils.ipaddr('address')
```

**Flow:** Ansible reads NetBox → builds inventory → runs playbooks against live infrastructure state.

**Idempotency:** Ansible inventory is always fresh from NetBox. No drift between inventory file and reality.

### 4.2 NetBox → Prometheus / Grafana

Each service in NetBox has a `prometheus_job` custom field. This links the NetBox service record to Prometheus scrape config.

**AI observability query flow:**
1. NetBox API → service definition + `prometheus_job` label
2. Prometheus API → current metrics (uptime, error rate, latency p95)
3. Grafana API → dashboard URL by service tag (`service:{name}`)
4. Loki API → recent errors for service label
5. Neo4J → downstream impact of active alerts

No hardcoded URLs — Grafana API is the live dashboard registry.

### 4.3 NetBox → Neo4J (Graph CMDB)

NetBox stores structured data (devices, IPs, services). Neo4J stores relationships (depends-on, hosted-on, connected-to).

**Sync pipeline:** NetBox API → Python/Ansible sync script → Neo4J (scheduled, idempotent)

```
MERGE (s:Service {slug: "gitlab"})
SET s.version = "17.0.1", s.lifecycle = "active"
MERGE (v:VM {name: "vm-gitlab-prod-01"})
MERGE (s)-[:HOSTED_ON]->(v)
```

> Neo4J `MERGE` is idempotent — creates if absent, updates if exists.

### 4.4 NetBox → Vault

Vault does not read from NetBox directly, but:
- NetBox custom field `repo_url` points to the repo where Vault paths are defined
- Vault secret paths follow the same slug convention: `secret/{scope}/{service}` (e.g. `secret/platform/gitlab`)
- Ansible reads NetBox inventory, then injects Vault secrets into service configs

### 4.5 NetBox → Terraform

Terraform provisions VMs on Proxmox. After provisioning:
- Ansible registers the new VM in NetBox (using `netbox.netbox.netbox_virtual_machine`)
- IP assignment is either pre-allocated in NetBox (IPAM) or registered post-provisioning

**Flow:** Terraform creates VM → Ansible registers in NetBox → NetBox becomes source of truth for that VM.

### 4.6 NetBox → K8S

K8S resources are not directly managed by NetBox, but:
- K8S namespace names match NetBox scope/env pattern: `platform-prod`, `mod-voip-prod`
- Service names in K8S match NetBox service slugs
- Grafana dashboards are tagged to match both NetBox and K8S labels

---

## 5. Idempotent Usage Strategies

### API-First Approach

All NetBox interactions should go through the REST API, not the web UI. This ensures:
- Reproducibility (API calls can be scripted and version-controlled)
- Auditability (API calls are logged)
- Idempotency (use `PUT` for updates, check-before-create patterns)

### Ansible Module Patterns

The `netbox.netbox` collection provides idempotent modules:

```yaml
# Ensure device exists (create or update)
- name: Register switch in NetBox
  netbox.netbox.netbox_device:
    netbox_url: "https://netbox.by-systems.internal"
    netbox_token: "{{ netbox_token }}"
    data:
      name: "sw-arista-prod-01"
      device_type: "7020TR-48Y"
      role: "switch"
      site: "brussels"
      status: "active"
      tags:
        - "tier-1"
        - "ansible-managed"
    state: present  # idempotent: creates if absent, updates if exists
```

```yaml
# Ensure IP prefix exists
- name: Register management prefix
  netbox.netbox.netbox_prefix:
    netbox_url: "https://netbox.by-systems.internal"
    netbox_token: "{{ netbox_token }}"
    data:
      prefix: "10.0.10.0/24"
      vlan:
        name: "mgmt"
      site: "brussels"
      status: "active"
      description: "Management network"
    state: present
```

### Tags for Automation Tracking

Use tags to track what manages each object:

| Tag | Meaning |
|---|---|
| `ansible-managed` | Object is created/updated by Ansible — do not edit manually |
| `terraform-managed` | Object is provisioned by Terraform |
| `manual` | Object was created manually (review for automation) |

### Custom Fields for Integration

Custom fields link NetBox objects to other systems without modifying the NetBox data model:

- `prometheus_job` → links to Prometheus scrape config
- `grafana_tag` → links to Grafana dashboard search
- `repo_url` → links to source code
- `version` → tracks deployed version (updated by CI/CD)
- `lifecycle` → tracks service lifecycle state

### Webhook-Driven Updates

NetBox webhooks can trigger automation on object changes:

| Event | Webhook target | Action |
|---|---|---|
| Device created | Ansible AWX/Semaphore | Run base hardening playbook |
| IP assigned | Bind 9 API | Update DNS record |
| Service version changed | Grafana API | Update dashboard annotation |
| Device decommissioned | Prometheus | Remove scrape target |

---

## 6. Backup and Recovery

| Component | Method | Schedule | Target |
|---|---|---|---|
| PostgreSQL (NetBox DB) | `pg_dump` | Daily | MinIO `s3://backups/netbox/` |
| Media files (uploads) | `rsync` or MinIO sync | Daily | MinIO `s3://backups/netbox-media/` |
| Configuration | Git (`platform-netbox-config`) | On change | GitLab |

### Recovery procedure

1. Restore PostgreSQL from latest `pg_dump`
2. Restore media files from MinIO
3. Verify API health: `GET /api/status/`
4. Run Ansible inventory test: `ansible-inventory -i inventory/netbox.yml --list`

---

## 7. NetBox Deployment

| Attribute | Value |
|---|---|
| Deployment | Docker Compose (Phase 1) → Helm chart (Phase 2+) |
| FQDN | `netbox.by-systems.internal` |
| Repo | `platform-netbox-config` |
| Database | Shared PostgreSQL cluster |
| Cache | Shared Redis |
| Reverse proxy | Traefik (TLS termination) |
| Auth | Authentik OIDC (via `django-auth-oidc` plugin or proxy auth) |
| Backup | Daily `pg_dump` → MinIO |

---

## 8. Naming Convention Alignment

NetBox object names must match the naming convention in `docs/naming-convention.md`:

| Domain | NetBox field | Convention | Example |
|---|---|---|---|
| Device name | `name` | `{function}-{vendor}-{env}-{number:02d}` | `sw-arista-prod-01` |
| VM name | `name` | `vm-{service}-{env}-{number:02d}` | `vm-gitlab-prod-01` |
| Service slug | `name` (service) | Component slug from repo name | `gitlab` |
| Tenant slug | `slug` | `{customer-short-name}` | `acme`, `rtbf` |
| FQDN | Computed | `{service}.by-systems.internal` | `netbox.by-systems.internal` |
| Prometheus job | Custom field | Matches service slug | `netbox` |

> **One slug, three systems:** A service named `gitlab` in NetBox = FQDN `gitlab.by-systems.internal` = repo `platform-gitlab-core`.
