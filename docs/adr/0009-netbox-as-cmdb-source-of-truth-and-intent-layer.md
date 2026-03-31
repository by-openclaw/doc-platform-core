# ADR 0009 — NetBox as CMDB Source of Truth and Intent Layer

**Status:** Accepted
**Date:** 2026-03-31
**Deciders:** @yboujraf
**Scope:** BY-SYSTEMS PoC platform — all infrastructure, not limited to compute

---

## Context

The platform requires a single source of truth for infrastructure state — covering not just servers and
switches, but every managed asset: NAS, firewalls, WiFi APs, PDUs, UPS, printers, TVs, mobile devices,
and logical objects derived from them (NAS shared folders, NFS rules, access permissions).

Without a unified CMDB:
- Root cause analysis (RCA) is impossible — no system knows the full dependency graph
- Ansible playbooks and IaC become isolated sources of partial truth
- Impact analysis requires manual correlation across tools
- There is no single answer to "what broke, why, and what else is affected"

**NetBox is not just an asset tracker.** It models:
- Physical devices (any device type — no "only servers" rule)
- Logical objects: interfaces, IPs, VLANs, prefixes, cables, power
- Custom objects (NetBox 4.x): NAS shares, NFS rules, group permissions, any domain-specific entity
- Relationships between all of the above

This ADR establishes NetBox as the **intent layer** (what should exist) and the **state layer** (what does
exist), with Ansible as the executor that reconciles intent to reality.

---

## Decision

**NetBox is the CMDB source of truth for all infrastructure — physical and logical.**

This means:

1. **Every managed asset is modeled in NetBox** — server, switch, NAS, firewall, WiFi AP, UPS, PDU,
   printer, TV, mobile, rack, AC unit. No exceptions based on device type.

2. **Logical objects derived from infrastructure are also modeled in NetBox** using custom objects:
   - NAS shared folders (name, path, protocol, quota)
   - NFS export rules (share → client IP/host, permissions)
   - Share access groups (share → Authentik group → permission level)
   - Any future domain-specific entity that belongs to the infrastructure graph

3. **NetBox is the intent source for Ansible** — a new share is created by modeling it in NetBox first,
   not by writing Ansible vars or editing a YAML file. Ansible reads NetBox (via dynamic inventory or
   webhook) and provisions accordingly.

4. **Ansible is the executor, not the source of truth.** Playbooks implement the "how". NetBox owns the
   "what" and "why". Ansible vars are derived from NetBox, not standalone.

5. **After provisioning, Ansible updates NetBox** with actual state (status field: provisioned/active/
   error). NetBox always reflects current reality, not just intent.

6. **NetBox syncs to Neo4j** (graph CMDB) to enable relationship traversal across the full topology:
   devices, services, identities (Authentik), shares, permissions, network paths.

7. **The agent (Rune) queries Neo4j** for RCA, blast radius analysis, and impact assessment. Natural
   language queries traverse the graph — "what would break if this NAS goes down?", "who has access to
   this share and via what path?", "what changed in the last 24h that could explain this failure?"

---

## Full Platform Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                        INTENT                                   │
│                                                                 │
│  Operator models object in NetBox:                              │
│    - New device → fill rack/IP/role                             │
│    - New NAS share → custom object (name, protocol, group ACL)  │
│    - New user group → Authentik (identity layer)                │
└────────────────────────┬────────────────────────────────────────┘
                         │ webhook / dynamic inventory
┌────────────────────────▼────────────────────────────────────────┐
│                        EXECUTION                                │
│                                                                 │
│  Ansible playbook triggered:                                    │
│    - Reads NetBox object (share name, quota, group)             │
│    - Reads Authentik group membership                           │
│    - Provisions NAS: ShareManager + NFSManager + ACLManager     │
│    - Updates NetBox status → "provisioned"                      │
└────────────────────────┬────────────────────────────────────────┘
                         │ state sync (ETL / NetBox→Neo4j plugin)
┌────────────────────────▼────────────────────────────────────────┐
│                        GRAPH CMDB (Neo4j)                       │
│                                                                 │
│  Nodes: Device, Share, Group, User, VM, Service, VLAN, IP       │
│  Edges: mounts, has_access, runs_on, connected_to, member_of    │
│                                                                 │
│  Example graph path:                                            │
│  VM-042 → runs_on → Proxmox-A → mounts → NAS-share:backup      │
│           → has_access → group:nas-proxmox → member → svc-tf   │
└────────────────────────┬────────────────────────────────────────┘
                         │ natural language query
┌────────────────────────▼────────────────────────────────────────┐
│                        AGENT (Rune)                             │
│                                                                 │
│  "VM-042 backup jobs failing since 02:00"                       │
│  → graph traversal: VM → Proxmox → NAS mount → NFS rule        │
│  → checks: last ACL change, NFS rule status, NAS health         │
│  → answer: "NFS rule for 10.6.224.105 removed 01:47 by         │
│    Ansible run job #212 — rollback or re-apply?"                │
└─────────────────────────────────────────────────────────────────┘
```

---

## Consequences

**Positive:**
- Single source of truth — no split-brain between Ansible vars and reality
- RCA becomes possible: graph traversal replaces manual tool correlation
- Impact analysis before changes: "blast radius" query on Neo4j
- NetBox models everything — no artificial "only compute devices" boundary
- Authentik groups are linked in the graph → identity-aware access reasoning
- Ansible becomes stateless executor — idempotent, driven by NetBox state

**Negative:**
- NetBox custom object schema must be designed and maintained per domain (NAS shares, etc.)
- NetBox→Neo4j ETL pipeline must be built and kept in sync
- Additional operational complexity: NetBox + Neo4j + Ansible all need to be running
- Team must discipline: intent goes into NetBox first, not directly into Ansible vars
- NetBox 4.x required for custom objects — verify version compatibility with DS1513+ timeline

## Deferred Decisions

- NetBox plugin vs custom object API for NAS shares — decide when NetBox CE is deployed
- Neo4j sync mechanism: custom ETL vs existing NetBox→Neo4j OSS connectors
- Agent query interface for Neo4j (Cypher vs natural language embedding layer)
- Authentik→NetBox group sync mechanism (webhook vs polling)

---

## References

- NetBox custom objects: <https://netboxlabs.com/docs/netbox/en/stable/customization/custom-objects/>
- NetBox dynamic inventory for Ansible: <https://github.com/netbox-community/ansible_modules>
- NetBox→Neo4j sync (community): <https://github.com/netbox-community/netbox-graph>
- ADR-0001 platform stack decisions
- ADR-0004 identity/SSO architecture (Authentik)
- lib-synology-dsm issues: #33 SystemManager, #38 SharePermissionManager
