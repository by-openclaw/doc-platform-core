# ADR 0009 — NetBox as Universal CMDB Source of Truth and Intent Layer

**Status:** Accepted
**Date:** 2026-03-31
**Deciders:** @yboujraf
**Scope:** BY-SYSTEMS PoC platform — ALL devices and software hosted in the infrastructure, no exceptions

---

## Context

The platform requires a single source of truth for infrastructure state — covering not just servers and
switches, but every managed asset and every piece of software running in the infrastructure.

Without a unified CMDB:
- Root cause analysis (RCA) is impossible — no system knows the full dependency graph
- Ansible playbooks and IaC become isolated sources of partial truth with no cross-tool visibility
- Impact analysis requires manual correlation across tools
- There is no single answer to "what broke, why, and what else is affected"

**NetBox is not just an asset tracker.** It models:
- Physical devices (any type — servers, switches, NAS, firewalls, WiFi APs, PDUs, UPS, printers, TVs,
  mobiles, racks, AC units — no "only servers" rule)
- Software and services hosted in the infrastructure (VMs, containers, applications)
- Logical objects: interfaces, IPs, VLANs, prefixes, cables, power feeds
- Custom objects (NetBox 4.x): any domain-specific entity — NAS shares, NFS rules, group permissions,
  service configs, application metadata
- Relationships between all of the above

**If a NetBox plugin does not exist for a domain, it must be developed** — minimum viable scope first,
extended iteratively. No domain is left unmodeled.

This ADR establishes NetBox as the **intent layer** (what should exist) and the **state layer** (what
does exist), with Ansible as the executor that reconciles intent to reality.

---

## Decision

**NetBox is the universal CMDB source of truth for all infrastructure — physical, logical, and software.**

This means:

1. **Every managed device is modeled in NetBox** — server, switch, NAS, firewall, WiFi AP, UPS, PDU,
   printer, TV, mobile, rack, AC unit. No exceptions based on device type.

2. **Every software system and service hosted in the infrastructure is modeled in NetBox** — VMs,
   containers, applications (Vault, Authentik, GitLab, NetBox itself, Wazuh, Prometheus, k3s workloads,
   etc.), service endpoints, configuration metadata.

3. **Logical objects derived from any system are modeled in NetBox** using custom objects:
   - NAS: shared folders, NFS rules, group ACLs, quotas
   - Proxmox: VM inventory, storage pools, cluster topology
   - Network: VLAN assignments, firewall zones, port mappings
   - Identity: service accounts, group→resource mappings
   - Any future domain — model it in NetBox first, not in a YAML file

4. **If no NetBox plugin exists for a domain, one must be developed:**
   - Phase 1 (minimum viable): list objects, CRUD, read-back actual state from target system
   - Phase 2: relationships, webhook triggers, Ansible dynamic inventory integration
   - Phase 3: Neo4j sync nodes/edges, agent-queryable metadata
   - Repository: `by-openclaw/netbox-plugin-<domain>` — follows BY-SYSTEMS naming convention
   - Language: Python (NetBox plugin standard) — zero runtime deps where possible
   - DoD: same as all BY-SYSTEMS repos — tested, linted, CHANGELOG, CI gate

5. **NetBox is the intent source for Ansible** — a new resource (share, VM, service, config) is created
   by modeling it in NetBox first. Ansible reads NetBox via dynamic inventory or webhook and provisions.
   No standalone Ansible vars files as source of truth.

6. **Ansible is the executor, not the source of truth.** Playbooks implement the "how". NetBox owns the
   "what" and "why". Ansible vars are derived from NetBox, not maintained in parallel.

7. **After provisioning, Ansible updates NetBox** with actual state (status: provisioned/active/error).
   NetBox always reflects current reality, not just intent.

8. **NetBox syncs to Neo4j** (graph CMDB) to enable relationship traversal across the full topology:
   devices, services, identities (Authentik), shares, permissions, network paths, dependencies.

9. **The agent (Rune) queries Neo4j** for RCA, blast radius analysis, and impact assessment. Natural
   language queries traverse the graph — "what would break if this NAS goes down?", "who has access to
   this share and via what path?", "what changed in the last 24h that could explain this failure?"

---

## Full Platform Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                        INTENT                                   │
│                                                                 │
│  Operator models object in NetBox:                              │
│    - New device    → fill rack/IP/role/vendor                   │
│    - New NAS share → custom object (name, protocol, group ACL)  │
│    - New VM        → virtual machine + service + IP             │
│    - New app       → service endpoint + config metadata         │
│    - New identity  → Authentik (identity layer, synced back)    │
└────────────────────────┬────────────────────────────────────────┘
                         │ webhook / dynamic inventory
┌────────────────────────▼────────────────────────────────────────┐
│                        EXECUTION (Ansible)                      │
│                                                                 │
│  Playbook triggered per resource type:                          │
│    - Reads NetBox object + Authentik group membership           │
│    - Provisions target: NAS / Proxmox / switch / app            │
│    - Updates NetBox status → "provisioned"                      │
└────────────────────────┬────────────────────────────────────────┘
                         │ state sync (ETL / NetBox→Neo4j)
┌────────────────────────▼────────────────────────────────────────┐
│                     GRAPH CMDB (Neo4j)                          │
│                                                                 │
│  Nodes: Device · Share · Group · User · VM · Service · VLAN     │
│  Edges: mounts · has_access · runs_on · connected_to ·          │
│         member_of · depends_on · exposes · manages              │
│                                                                 │
│  Example path:                                                  │
│  VM-042 → runs_on → Proxmox-A → mounts → NAS:proxmox-backup    │
│        → has_access → group:nas-proxmox → member_of → svc-tf   │
└────────────────────────┬────────────────────────────────────────┘
                         │ natural language query
┌────────────────────────▼────────────────────────────────────────┐
│                        AGENT (Rune)                             │
│                                                                 │
│  "VM-042 backup jobs failing since 02:00"                       │
│  → graph traversal: VM → Proxmox → NAS mount → NFS rule        │
│  → checks: last ACL change, NFS rule status, NAS health         │
│  → "NFS rule for 10.6.224.105 removed 01:47 by Ansible          │
│     job #212 — rollback or re-apply?"                           │
└─────────────────────────────────────────────────────────────────┘
```

---

## Consequences

**Positive:**
- Single source of truth — no split-brain between Ansible vars and reality
- RCA becomes possible: graph traversal replaces manual cross-tool correlation
- Blast radius analysis before any change — query Neo4j, not gut instinct
- NetBox models everything — no artificial "only compute devices" boundary
- Authentik groups linked in the graph → identity-aware access reasoning
- Ansible becomes stateless executor — idempotent, fully driven by NetBox state
- Any new domain gets the same treatment: model in NetBox, automate with Ansible, query via agent

**Negative:**
- NetBox custom object schemas must be designed and maintained per domain
- Missing plugins must be developed in-house — additional repo + maintenance burden per domain
- NetBox→Neo4j ETL pipeline must be built and kept in sync
- Operational complexity: NetBox + Neo4j + Ansible must all be healthy for the loop to work
- Team discipline: intent goes into NetBox first, never directly into Ansible vars or ad-hoc scripts
- NetBox 4.x required for custom objects — verify version on deployment
- Plugin development effort must be time-boxed: minimum viable first, no gold-plating phase 1

---

## Plugin Development Pipeline

When a NetBox plugin is required but does not exist, follow this pipeline:

| Phase | Scope | Deliverable |
|---|---|---|
| 1 — Minimum viable | List + CRUD + read-back actual state | Working plugin, installable, CI green |
| 2 — Integration | Relationships, webhooks, Ansible dynamic inventory | Ansible reads NetBox for this domain |
| 3 — Graph | Neo4j sync nodes + edges, agent metadata | Agent can reason about this domain |

**No phase is skipped. No phase 3 without a working phase 1.**

---

## Deferred Decisions

- Priority order for plugin development (NAS first → Proxmox → network → ...)
- NetBox plugin vs custom object API per domain — evaluate at deployment time per domain
- Neo4j sync mechanism: custom ETL vs existing OSS connectors (netbox-graph or equivalent)
- Agent query interface for Neo4j: Cypher direct vs natural language embedding layer
- Authentik→NetBox group sync: webhook push vs polling
- Which services get modeled as NetBox VirtualMachines vs custom objects

---

## CISO mapping

> Applies only to controls directly relevant to this ADR's scope.

### ISO/IEC 27001:2022

| Control | Title | Status | Notes |
|---|---|---|---|
| A.8.1 | Inventory of assets | ✓ Covered | NetBox is the CMDB source of truth for all assets — physical and logical |
| A.8.9 | Configuration management | ⚠ Partial | Config state modeled in NetBox; automation enforcement pending Ansible integration |
| A.5.9 | Inventory of information and other assets | ✓ Covered | NetBox models physical devices, VMs, services, VLANs, IPs |

### NIS2 (Directive 2022/2555)

| Article | Requirement | Status | Notes |
|---|---|---|---|
| Art. 21(2)(a) | Risk management policies | ⚠ Partial | Asset visibility is required for risk assessment; NetBox provides it once fully populated |
| Art. 21(2)(f) | Policies on the use of cryptography | ✓ Covered | NetBox models network topology enabling cryptographic segmentation validation |

### GDPR (Regulation 2016/679)

Not applicable — NetBox models infrastructure assets, not personal data.

## References

- [NetBox custom objects](https://netboxlabs.com/docs/netbox/en/stable/customization/custom-objects/)
- [NetBox dynamic inventory for Ansible](https://github.com/netbox-community/ansible_modules)
- [NetBox→Neo4j sync (community)](https://github.com/netbox-community/netbox-graph)
- [lib-synology-dsm GitHub issues #33 (SystemManager) and #38 (SharePermissionManager)](https://github.com/by-openclaw/lib-synology-dsm/issues)
