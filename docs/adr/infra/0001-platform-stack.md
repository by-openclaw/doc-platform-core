# infra/0001 — Platform Stack

**Status:** Draft
**Date:** 2026-04-13 (supersedes flat ADR-0001, 2026-03-25)
**Scope:** Tool inventory for the BY-SYSTEMS platform. This ADR lists **what** tools are chosen. Decisions about **how** they are used live in specialized scoped ADRs.
**Related:** `infra/0002-platform-charter`, `infra/0003-terraform-standard`, `infra/0004-network-architecture`, `git/0002-platform-strategy`, `identity/*`, `naming/*`, `security/*`

---

## Context

BY-SYSTEMS is a broadcast / telecom system integrator building a self-hosted DevOps platform. Goals:

- Replace legacy and manual workflows with proper DevOps tooling
- Reference architecture for SME customers
- Support broadcast, VoIP, and CCTV integration projects
- All tooling open source, self-hosted, free tier — no vendor lock-in

This ADR is the **top-level platform stack inventory**. Other scoped ADRs own the actual decisions (workflow, naming, identity model, certificate strategy, secret storage, etc.). This one is a pointer table for onboarding and audit.

Flat ADR-0001 (the source) was written as a bootstrap document before proper ADRs existed and accumulated 11 sections' worth of content. Most of that content has been split into the scopes that now own each concern. This shrunk replacement keeps only the **tool choices** that are not owned by a more specific scoped ADR.

## Decision

### 1. Compute & Virtualisation

| Tool | Role | License |
|---|---|---|
| Proxmox VE | Hypervisor (VM + LXC) | AGPL v3 |
| k3s | Lightweight Kubernetes distribution | Apache 2.0 |
| k9s + Headlamp | Kubernetes operations (CLI + web UI) | Apache 2.0 / Apache 2.0 |
| Kaniko | Rootless container image builds in CI | Apache 2.0 |
| Packer | Golden image builds | MPL 2.0 |

**Rationale:**
- Proxmox replaces VMware — zero licensing cost, KVM + LXC support, active community
- k3s is the lightest production-ready Kubernetes — appropriate for SME scale without full k8s overhead
- Kaniko enables rootless CI image builds — no Docker daemon required
- Kubernetes migration path: all Layer 5 tools ship with both Docker Compose and Helm deployment options — start with Docker, switch to k3s when load justifies it

### 2. Infrastructure as Code

| Tool | Role | License |
|---|---|---|
| Terraform | Declarative provisioning via Proxmox provider | MPL 2.0 |
| Ansible | Configuration management + network automation | GPL v3 |
| `community.general` / `arista.eos` / `netbox.netbox` Ansible collections | OS config, Arista EOS switches, NetBox dynamic inventory | GPL v3 / Apache 2.0 / MIT |

**Rationale:**
- Terraform provisions (VM lifecycle), Ansible configures (post-boot, network device config) — idiomatic pairing
- `netbox.netbox` collection gives Ansible dynamic inventory from NetBox, so the inventory is the NetBox device list
- `arista.eos` collection replaces hand-written switch configs with declarative tasks

Terraform state management, backend, and provider constraints are in `infra/0003-terraform-standard`.

### 3. Storage & Databases

| Tool | Role | License |
|---|---|---|
| PostgreSQL (Patroni) | Primary relational DB for platform tools (GitLab, Vault, NetBox, Authentik, Grafana) | PostgreSQL License / Apache 2.0 |
| Redis (Sentinel) | Cache, sessions, queues | BSD 3-Clause / BSD 3-Clause |
| MinIO | S3-compatible object storage (Loki backend, backups, CI artifacts) | AGPL v3 |
| Neo4J Community | Graph CMDB — dependency and relationship mapping | GPL v3 |

**Strategy:** start single-instance, scale to cluster when load justifies it. Patroni for PostgreSQL HA, Sentinel for Redis HA.

**Why Neo4J alongside NetBox:** NetBox owns the structured source of truth (devices, VLANs, IPAM, circuits). Neo4J adds the graph layer that NetBox's relational model cannot express — dependency chains, RCA traversal, impact analysis. Sync direction: NetBox → Neo4J, event-driven (webhooks).

Database per-service deployment, backup, and operational rules are in `infra/0008-backup-strategy` and per-service runbooks.

### 4. Documentation

| Tool | Role | License |
|---|---|---|
| Markdown + PlantUML | Source format for ADRs, runbooks, standards | Various / MIT-like |
| Kroki | Self-hosted diagram renderer (PlantUML, Mermaid, D3, BPMN, Graphviz) | MIT |
| Sphinx | Build engine — multi-format output (PDF, HTML, DOCX) | BSD 2-Clause |
| Draw.io (desktop) | Architecture diagrams authored outside Git | Apache 2.0 |

**Rationale:**
- **Docs-as-code** — all documentation lives in Git, versioned alongside the infrastructure it describes
- **Kroki** renders PlantUML, Mermaid, D3, BPMN, Graphviz from one self-hosted container — no external renderer dependency, no SaaS call
- **Sphinx** provides professional multi-format output (PDF, HTML, DOCX) for customer-facing deliverables
- **Draw.io** is allowed for complex architecture diagrams that don't render well in text-based formats; source `.drawio` files are committed alongside the rendered export

**Rules:**
- ADRs are always Markdown (no Draw.io-only diagrams in ADR content)
- Every diagram has both source and rendered output committed
- No Notion / Confluence / GitBook — SaaS documentation tools are explicitly out of scope

### 5. Cross-reference — decisions owned by other scopes

The following concerns are intentionally NOT in this ADR. Each scope owns its own set of decisions; this table is grouped by scope for navigation.

#### `git/`

| ADR | Concern |
|---|---|
| `git/0001-workflow` | Trunk-based branching, Conventional Commits, PR flow, merge strategy, agent boundary |
| `git/0002-platform-strategy` | GitHub Free (private repos, current) → GitLab CE self-hosted (target), CI platform |
| `git/0003-configuration` | `/etc/gitconfig` + `~/.gitconfig` templates, cross-OS (SSH/WinRM) applicability |

#### `identity/`

| ADR | Concern |
|---|---|
| `identity/0001-authentication` | Authentik as IAM hub, 3-tier identity model, optional federation, fallback behavior |
| `identity/0002-provisioning` | Authentik → tools sync via Ansible `identity-sync` role + per-tool adapters |
| `identity/0003-machine-credentials` | `{IDENTITY}_{PLATFORM}_TOKEN` naming, PAT vs secret storage, Free plan rules |
| `identity/0004-os-accounts` | Linux host accounts, SSH/GPG keys, sudo, break-glass (OOB-CIDR password auth) |

#### `naming/`

| ADR | Concern |
|---|---|
| `naming/0001-infra` | Hostname pattern `{site}-{service}-{seq}`, NetBox as SoT, dual-stack DNS, env sub-domain |
| `naming/0002-identity` | Human / service / temporary account patterns, OS groups, Authentik groups |
| `naming/0003-firewall` | OPNsense alias naming (`net_*`, `host_*`, `port_*`, `grp_*`), rule description format |
| `naming/0004-automation` | Repo, Python, Ansible, Terraform naming + domain abbreviation table |

#### `security/`

| ADR | Concern |
|---|---|
| `security/0001-secret-storage` | HashiCorp Vault (machine) + Vaultwarden (human), KV path `secret/{env}/{service}/{key}` |
| `security/0002-compliance-mapping` | ciso-assistant-community as the authoritative compliance registry |
| `security/0003-hardening` | Non-root, image hygiene, Trivy / Lynis, patch cadence, cross-OS note |
| `security/0004-certificate-strategy` | Let's Encrypt (public) + step-ca (internal), Traefik as termination point |

#### `infra/` (this scope)

| ADR | Concern |
|---|---|
| `infra/0002-platform-charter` | Layer model — Layer 0 Standards → Layer 5 Platform Services, gating rules |
| `infra/0003-terraform-standard` | State management (NAS now → GitLab backend target), provider constraints |
| `infra/0004-network-architecture` | Network zones, VLAN registry, Proxmox SDN, WireGuard peer naming |
| `infra/0005-environment-tiers` | `dev` / `test` / `staging` / `acc` / `prod` / `drp` (6 tiers, no `poc`) |
| `infra/0006-logging` | Loki + Promtail, S3 backend on Contabo, retention, no-PII rule |
| `infra/0007-monitoring` | Prometheus + Grafana, `/metrics` scrape requirement, alert baseline |
| `infra/0008-backup-strategy` | Per-service-class backup policy, Synology NAS + off-site, restore SLA |

#### `services/` (future refactor)

| ADR | Concern |
|---|---|
| `services/0001-opnsense-provisioning` | OPNsense API-first provisioning, lib-opnsense + ansible-opnsense |
| `services/0002-email-infrastructure` | Mailcow deployment, MX routing, DMARC / DKIM / SPF, on Contabo |
| `services/0003-netbox-cmdb` | NetBox as CMDB source of truth, role vocabulary, Ansible dynamic inventory |
| `services/0004-database-strategy` | PostgreSQL + Redis operational strategy (single-instance → Patroni/Sentinel) |
| `services/0005-licensing-policy` | Open-source licensing policy for all self-published code |

#### `lib/` (future refactor)

| ADR | Concern |
|---|---|
| `lib/python/0001-design-standard` | Python library design — client/manager/model/exception class hierarchy, DI pattern, testing contract |

## Consequences

- **This ADR is a pointer table, not a decision document.** Every tool listed here is a tool choice; every detail about how the tool is used lives in a specialized ADR.
- **Onboarding** — new engineers and agents can read this one file to see the full tool inventory, then follow the cross-reference table to the specific decisions that interest them.
- **Audit** — auditors have a single top-level architecture reference that lists every tool, its role, and its license.
- **Drift risk is low** because this ADR delegates. When a tool is replaced or a decision changes, only the specialized ADR needs updating — this file updates only when a tool is added to or removed from the platform entirely.
- **Full OSS stack** — zero licensing cost across compute, IaC, storage, documentation. Self-hosted, no SaaS dependencies in normal operation (Cloudflare DNS-01 is the only external dependency, and it is used only for TLS cert issuance via ACME).

## Revision triggers

Revise this ADR when:
- A tool in §1 / §2 / §3 / §4 is replaced (e.g. Terraform → OpenTofu, PostgreSQL → CockroachDB)
- A tool is added that does not fit under any existing scoped ADR
- A new scope folder is added to `docs/adr/` that should appear in §5
- The cross-reference table drifts from the actual ADR file paths

## CISO mapping

| Framework | Controls covered |
|---|---|
| ISO 27001:2022 | A.5.23 (use of cloud services — fully self-hosted), A.8.9 (configuration management — tool inventory is versioned) |
| NIS2 | Art. 21(2)(a) (risk management — known tools, known licenses, no surprise dependencies) |
