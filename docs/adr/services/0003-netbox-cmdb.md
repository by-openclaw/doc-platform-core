# services/0003 — NetBox CMDB

**Status:** Draft (rewritten 2026-10-31 against the running platform — proposal, approve by merging)
**Date:** 2026-04-14 (supersedes flat ADR-0009, 2026-03-31); rewritten 2026-10-31
**Scope:** How NetBox works as the platform CMDB — what it models (physical, virtual, logical, services), the mandatory fields that are the contract with every other ADR, the closed role vocabulary, how data gets in (bootstrap import, our own collection library, plugins), who reads it (Ansible, Terraform, monitoring, the register), the lifecycle that lets hardware go in and out of the rack, and the build plan from the empty instance that exists today. Does **not** re-assert "NetBox is the source of truth" (`naming/0001-infra §1`) nor define hostnames (`naming/0001-infra §3–5`).
**Related:** `naming/0001-infra`, `infra/0001-platform-stack`, `infra/0003-terraform-standard`, `infra/0004-network-architecture`, `infra/0005-environment-tiers`, `infra/0007-monitoring`, `infra/0008-backup-strategy`, `infra/0010-provisioning-ansible-only`, `security/0001-secret-storage`, `services/0004-database-strategy`

---

## Context

- NetBox **runs** (`lxc-nbox-01`, NetBox 4.7, Authentik OIDC, shared PostgreSQL and Redis, PBS backup, metrics scraped). Its data model is **empty**: on 2026-10-31 every object class — sites, racks, devices, device types, interfaces, cables, power, VLANs, prefixes, addresses, circuits, VMs, custom fields, tags, webhooks — counted zero. The role deploys the application; nothing seeds the model.
- The platform's facts live in four other places: `inventories/<env>/hosts.yml` (guests, addresses, groups), `register.yml` → `docs/register.md` (the 34 services with exposure, SSO, database, backup, monitoring, owner), the Terraform environment (guests, SDN VLANs, prefixes) and the firewall catalog (aliases, rules, VLAN interfaces). None of them knows a rack unit, a cable, a power feed or an ISP circuit.
- The owner decided on 2026-09-02/03: NetBox is inventory-only (metrics stay in Prometheus/Grafana, each device deep-links to its dashboard); the model covers **everything** in the DevOps infrastructure (network, compute, storage, power, racks, cabling of every medium, modules, connectors); collection goes through a **single owned library** pushing into NetBox via Diode, no third-party discovery sprawl; physical switching is Arista (core and PoE campus) with the interim Netgear disappearing and Cisco Nexus for interop.
- The previous text of this ADR carried placeholder names (`vm-nbox-01`, Pi-hole, MinIO, WireGuard, a graph database) that never matched the platform. This rewrite replaces them with the running estate and removes what has no owner decision behind it.

## Decision

### 1. Data model — what NetBox holds

**Everything managed is in NetBox; if it is not in NetBox it is not managed.** Secrets, metrics, logs and audit trails stay where their ADRs put them (Vault, Prometheus, Loki); NetBox may reference a Vault **path**, never a value.

#### 1.1 Physical (site `br`, naming/0001 §4)

| Object | NetBox class | Today | Source for the load |
|---|---|---|---|
| Site, location | `Site` `br`, `Location` for each room (the basement rack room first) | 1 site | owner |
| Racks | `Rack` with height, width, role, reservation for planned gear | 1 rack room, racks counted on the pictures | owner's front/rear photographs (2026-10-31) |
| Servers | `Device` — `srv-proxmox-poc-01` (HPE ProLiant DL380p Gen8, the live platform host), `pve01` (DL380p Gen8, production host: OOB gateway, PTP grandmaster), `br-pmox-03` (Tyan GT24-B8236-IL), the fourth DL380p Gen8 once it powers on | 3 + 1 | `dmidecode`, iLO/IPMI, Terraform, Proxmox API |
| Storage | `Device` — Synology DS1513+ (`nas01`) | 1 | DSM |
| Network | `Device` — interim Netgear GS switch(es), Arista core and PoE campus when installed, UniFi access points, ISP CPE (Telenet, Proximus) | from the photographs | owner, LLDP via the collection library |
| Power | `PowerPanel`, `PowerFeed`, PDU and UPS as `Device` with power ports and outlets | unknown — read from the photographs | photographs, UPS/PDU API |
| Passive | patch panels as `Device` with `FrontPort`/`RearPort`, cable trays as `Location` notes | from the photographs | photographs |
| Cables | `Cable` for **every** medium — copper, fibre, serial console, power, video, audio — with type, colour, length, label; modules (`ModuleType` SFP/QSFP) in `ModuleBay`s | 0 | photographs, LLDP, rewiring day |
| Circuits | `Provider` + `Circuit` + terminations: Telenet (static anchor), Proximus PPPoE (dynamic anchor); the off-site object store and PBS target as a cloud `Site` | 0 | firewall seed, contracts |
| Management | native `Device.oob_ip` for iLO/IPMI/BMC addresses, `console-port` for serial | 0 | iLO, IPMI |

Device types come from the public `netbox-community/devicetype-library` where they exist (HPE ProLiant DL380p Gen8 with front and rear elevation images, Netgear GS, Arista, Ubiquiti U6, APC and Eaton power gear, Synology DS15xx siblings); the **DS1513+** and the **Tyan GT24-B8236-IL** are missing there and are authored by us in the same YAML format (and contributed upstream).

Passive and physical assets get names under the `naming/0001` pattern: `br-rk-NN` rack, `br-pp-NN` patch panel, `br-pdu-NN`, `br-ups-NN`, `br-sw-NN` switch, `br-ap-NN` access point, `br-cpe-<isp>-01` ISP equipment. These rows are mirrored into `naming/0001 §5` in the companion PR on approval.

#### 1.2 Virtual

`Cluster` per hypervisor (the three nodes, the fourth on arrival; cluster type `proxmox`), one `VirtualMachine` per VM **and** per LXC (NetBox has no container class; the custom field `guest_type` = `vm`/`lxc` carries it), with vCPU, memory, disk, interfaces, addresses and the service fields of §2. Kubernetes nodes (`vm-k3s-01`) are VMs; cloud guests (`cb-*`) sit in the cloud `Site`.

#### 1.3 Logical

VLANs as a `VLANGroup` per fabric — the production SDN set (`mgmt` 1010, `dmz` 1020, `svc` 1030, `vpn` 1040, `iot` 1100, `voip` 1110, `storage` 1200, `media` 1300, `cctv` 1400), the out-of-band and fabric-management VLANs, the ISP-facing bridges — with their IPv4 **and** IPv6 prefixes (`naming/0001 §6.1`: dual-stack is mandatory), every guest address, the firewall's interface addresses and the VRRP/virtual addresses (resolver VIP). The supernets and ULA prefixes are imported from the Terraform SDN definition and the inventory, not typed by hand.

#### 1.4 Services

One `Service` object per platform service bound to its guest(s) with port and protocol, carrying the fields of §2. **This replaces `register.yml` as the home of the register:** `docs/register.md` is generated from NetBox after the load (Phase 1 below) and `register.yml` is archived.

### 2. Mandatory custom fields — the contract with the other ADRs

The fields are the columns the register already enforces today, so the migration changes the store, not the contract.

| Field | On | Type | Values / meaning | Consumed by |
|---|---|---|---|---|
| `env` | Device, VM | choice | `dev` `test` `staging` `acc` `prod` `drp` (`infra/0005`) | Vault path, DNS zone, inventory group |
| `guest_type` | VM | choice | `vm` / `lxc` | Terraform module, swap and storage standards |
| `service` | VM, Service | choice (closed list = §3) | which platform service this asset belongs to | playbook selection, naming short code |
| `exposure` | Service | choice | `public` / `private` / `admin` | DNS, Traefik, Authentik groups, NetBird policy, host firewall, JumpServer grants |
| `sso` | Service | choice | `oidc` / `forwardauth` / `ldap` / `local` / `none` | identity ADRs, audit |
| `db` | Service | choice | `shared-pg` / `own` / `none` | `services/0004` consumer table |
| `backup` | Service, Device | multi-choice | `pbs` `app-native` `pg-dump` `s3` `replica` `config-export` `nfs` `raft-snapshot` | `infra/0008` jobs and restore drills |
| `monitoring` | Service, Device | multi-choice | `node` `cadvisor` `probe` `metrics` | `infra/0007` scrape and probe generation |
| `service_url` | Service | URL | user-facing HTTPS entry | DNS, Traefik, blackbox probe |
| `image` | Service | text | pinned image or package version (`never :latest`) | update review, CVE matching |
| `vault_path_prefix` | VM, Service | text | `<env>/<service>` | `security/0001` |
| `owner` | Device, VM, Service | contact | responsible person | incident response |
| `doc_url` | Service | URL | vendor documentation the configuration follows | vendor-conformance reviews |
| `grafana_url` | Device, VM | URL | deep link to the asset's dashboard (owner 2026-09-03: inventory-only, metrics elsewhere) | people |

Adding or renaming a field is an amendment of this ADR; the fields are created by the `cmdb` playbook, never in the UI.

### 3. Canonical vocabulary

`service` is a closed list and equals the register's service keys as they run today:

| Class | Values |
|---|---|
| Core / identity | `opnsense` `adguard` `step-ca` `vault` `authentik` `netbird` `traefik` `crowdsec` `warden` `wazuh` |
| Data | `postgresql` `dbproxy` `redis` `seaweedfs` `pbs` |
| Platform | `proxmox` `k3s` `portainer` `monitoring` `netbox` `pgadmin` `jumpserver` `pkgcache` |
| Delivery | `gitlab` `gitlab-runner` `harbor` `verdaccio` |
| Collaboration | `nextcloud` `collab` `jitsi` `mailcow` `vaultwarden` `diagrams` `grc` |

Device roles (`DeviceRole`) for physical assets: `hypervisor` `nas` `switch` `access-point` `isp-cpe` `pdu` `ups` `patch-panel` `workstation`. Virtual roles reuse the `service` value. New values enter by amendment of this ADR; the UI cannot add them (the playbook reconciles the choice sets and removes strays).

### 4. How data enters NetBox

| Writer | What | How |
|---|---|---|
| **Bootstrap import** (once per environment) | sites, racks, device types, devices, VLANs, prefixes, guests, services | `cmdb` playbook with the `netbox.netbox` collection reading today's sources: Terraform state, `hosts.yml`, `register.yml`, the firewall catalog, Proxmox and iLO/IPMI APIs, the photographs for rack units and cabling |
| **Operator** | intent before provisioning: a planned device in a reserved rack unit, a new VLAN, a new service | pull request on the `cmdb` playbook data (reviewed like any change), applied by the playbook — the UI is read and browse |
| **Collection library** (owned, like `lib-opnsense`) | discovered state: LLDP neighbours, interface states, optics, serials, firmware, UPS load | minimal protocols (SNMP, SSH, vendor APIs: eAPI, NX-API, UniFi, OPNsense, iLO/Redfish, IPMI, DSM) → Diode → NetBox; no Oxidized, Orb or NAPALM |
| **Provisioning** | status transitions and last-check timestamps | post-tasks of the service playbooks (`infra/0010`) |
| **Plugins** | domains the stock model lacks (NAS shares and exports, firewall alias registry, AV signal paths) | `netbox-plugin-<domain>`, three phases: minimum viable → integration (webhooks, inventory) → graph/agent; no phase skipped |

Drift (discovered ≠ intent) is logged to Loki and flagged on the object; the playbook never silently overwrites intent.

### 5. Who reads NetBox

| Consumer | Mechanism | Phase |
|---|---|---|
| Ansible | `netbox.netbox.nb_inventory` dynamic inventory, groups from `env`, `service`, `exposure`, cluster, site; `hosts.yml` kept beside it until parity, then archived | 3 |
| Register | `docs/register.md` generated from NetBox `Service` objects | 1 |
| Terraform | `terraform-provider-netbox` data sources for addresses, VLANs, device facts; resources for what Terraform creates | 3 |
| Monitoring | scrape and probe targets generated from `monitoring` and `service_url` | 3 |
| Firewall catalog | host and network aliases generated from prefixes and addresses | 4 |
| Switch configuration | vendor-neutral intent (ports, VLANs, LAGs, PoE, cabling) rendered per vendor (EOS, NX-OS) | 4 |
| People | the rack elevation, the cable list, the patch map, the deep links | 2 |

### 6. Lifecycle — hardware in and out of the rack

NetBox's native device statuses are the state machine; each transition is a playbook, not a hand edit.

| Status | Meaning | Playbook effect |
|---|---|---|
| `planned` | ordered or on the bench; a rack unit is reserved, addresses allocated from NetBox | nothing runs; DNS and firewall prepared |
| `staged` | racked and cabled per the model, management interface reachable | adoption: BMC credential to Vault, firmware baseline, hypervisor install or NAS join |
| `active` | in production | normal operation, monitoring, backup |
| `decommissioning` | leaving: data migrated | monitoring silenced, backup job retired after the last restore test, DNS and firewall entries removed, addresses released |
| `offline` / `inventory` | out of the rack, kept | record archived, never deleted (`feedback: never delete, archive`); the rack unit is free |

A proof-of-concept machine goes `planned → staged → active → decommissioning → inventory` in days; the model stays correct at each step because the playbooks refuse to act on a status they do not expect.

### 7. NetBox deployment parameters (as running)

| Parameter | Value |
|---|---|
| Hostname | `lxc-nbox-01` (`naming/0001 §5`, short code `nbox`) |
| Service URL | `netbox.{domain}` per `naming/0001 §7`; exposure `private` |
| Version | NetBox 4.7 line, image pinned by the role (`never :latest`) |
| Database, cache | shared PostgreSQL and Redis (`services/0004`), own database and ACL login |
| Media | local media volume backed up by PBS; moves to the platform object store when it grows past the volume class |
| Authentication | Authentik OIDC; break-glass local superuser and API token reconciled from `people.yml`, values in Vault (`security/0001`) |
| Automation identity | one API token per writer (playbook, collection library, Diode), scoped, in Vault, rotated with the platform rotation |
| Own record | NetBox is itself a `Service` (`service = netbox`) on `lxc-nbox-01` |

### 8. Deferred decisions

| Decision | Deferred until |
|---|---|
| Graph projection (Neo4j or other) for multi-hop impact queries | the model is loaded and a consumer needs it; not before Phase 3 |
| Order of plugins (NAS shares, firewall aliases, AV paths) | the first gap met after Phase 2 |
| Diode self-hosted versus library direct writes | the collection library's first discovery flow |
| Authentik → NetBox group sync: push or poll | the identity sync framework |

## Build plan (from the empty instance)

| Phase | Deliverable | Done when |
|---|---|---|
| 0 | this ADR and the `naming/0001` rows approved | merged |
| 1 | `cmdb` playbook: custom fields, choice sets, tags, device roles, manufacturers and device types (library import + DS1513+ and GT24 authored), VLAN groups, prefixes, clusters; services and guests imported from `register.yml`, `hosts.yml`, Terraform and Proxmox; `register.md` generated from NetBox | every object class non-zero, register diff against today's file is empty |
| 2 | physical load: racks, rack units, devices, power, patch panels, cables, circuits — from the photographs and iLO/IPMI reads; elevation and cable list printed | the printed elevation matches the rack; every cable has a label |
| 3 | consumers: `nb_inventory` at parity with `hosts.yml`, Terraform reading addresses from NetBox, monitoring targets generated | one playbook run on each inventory gives identical results; `hosts.yml` archived |
| 4 | rewiring day driven by the model; collection library first flows (LLDP, optics, serials); firewall aliases and switch intent rendered from NetBox | drift report clean after the rewiring |

Each phase is a series of pull requests verified by reading the NetBox API back, never by the UI.

## Consequences

- The register, the inventory and the rack become one dataset with one editing path (pull request → playbook); the UI is for reading.
- The vocabulary is closed and matches what runs; a new service exists only after this ADR has a row for it.
- Hardware changes for proofs of concept cost a status change, not a hunt through five files.
- Two device types and one collection library are our own code to maintain, with tests and CI like every platform repository.

## Revision triggers

- A new service, field or device role (§2, §3).
- The fourth hypervisor arrives (cluster rows, rack units) — data, not an amendment, unless the model changes.
- Arista core or PoE campus installed (§5 switch configuration becomes live).
- NetBox 5.x changes the data model.
- A consumer is added to §5.

## CISO mapping

| Framework | Controls covered |
|---|---|
| ISO 27001:2022 | A.5.9 (inventory of information and other assets — every asset, physical and virtual, in one register), A.5.12 (classification — `exposure`), A.7.4 (physical security monitoring — rack units and cabling known), A.8.9 (configuration management — mandatory fields are the contract), A.8.20 (networks security — VLANs, prefixes and cabling modelled), A.5.37 (documented operating procedures — lifecycle playbooks) |
| NIS2 | Art. 21(2)(a) (risk analysis — asset visibility is the prerequisite), Art. 21(2)(e) (security in acquisition, development and maintenance — lifecycle states), Art. 21(2)(i) (asset management) |
