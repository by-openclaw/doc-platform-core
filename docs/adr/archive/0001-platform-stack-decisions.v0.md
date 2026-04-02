# ADR 0001 — Platform Stack Decisions

**Date:** 2026-03-25  
**Status:** Accepted  
**Scope:** BY-SYSTEMS internal DevOps PoC platform (Tier 1 base + Tier 2 optional modules)
**Authors:** BY-SYSTEMS platform team  
**Related docs:** `docs/stack.md`, `docs/naming-convention.md`, `docs/archive/brainstorm-2026-03-25.md`

---

## Context

BY-SYSTEMS is a broadcast/telecom system integrator building an internal DevOps PoC platform. Goals:

- Replace legacy/manual workflows with proper DevOps tooling
- Serve as reference architecture for SME customers
- Support broadcast, VoIP, and CCTV integration projects
- All tooling: open source, self-hosted, free tier — no vendor lock-in

The platform is structured in two tiers:

- **Tier 1:** Base platform, required for all deployments
- **Tier 2:** Optional modules per customer profile (`module-broadcast`, `module-voip`, `module-cctv`)

Reference prior art: CLIENT-A CMDB audit (2024) — production app landscape, CI/CD pipeline, infrastructure, and Neo4J-backed CMDB.

---

## Decision

The following toolchain decisions are locked for the PoC phase. All tools are open source, self-hosted, and carry no mandatory licensing cost.

---

## 1. Compute & Virtualisation

### Decision: Proxmox VE + k3s

| Tool | Role |
|---|---|
| Proxmox VE | Hypervisor (VM + LXC) |
| k3s | Lightweight Kubernetes |
| k9s + Headlamp | K8S operations (CLI + web UI) |
| Kaniko | Rootless container image builds in CI |
| Packer | Golden image builds |

**Rationale:**
- Proxmox replaces VMware — AGPL v3, zero licensing cost, mature KVM + LXC support, active community
- k3s is the lightest production-ready Kubernetes distribution; appropriate for lab/SME scale without the overhead of full k8s
- Headlamp chosen over Rancher (OSS, lightweight, no opinionated cluster management layer)
- Kaniko enables rootless container builds inside Kubernetes CI pipelines — no Docker daemon required

**Alternatives considered:**
- **VMware vSphere**: Licensing cost, vendor lock-in — rejected
- **Rancher**: Adds unnecessary management layer over k3s — replaced by k9s + Headlamp
- **Full upstream Kubernetes (kubeadm)**: Overkill for SME PoC scale — k3s is sufficient

---

## 2. Infrastructure as Code

### Decision: Terraform (Proxmox provider) + Ansible

| Tool | Role |
|---|---|
| Terraform | VM/LXC provisioning via Proxmox provider |
| Ansible | Configuration management, network automation |
| `arista.eos` collection | Arista switch config via eAPI |
| `netbox.netbox` collection | NetBox dynamic inventory |

**Rationale:**
- Terraform handles declarative infrastructure lifecycle (create, update, destroy)
- Ansible is the de-facto standard for configuration management and network automation — direct support for Arista EOS and NetBox inventory
- Pairing is idiomatic: Terraform provisions, Ansible configures

**Alternatives considered:**
- **Pulumi**: More code-centric but adds complexity and weaker community support for Proxmox — rejected for PoC
- **Chef/Puppet**: Legacy tooling, heavier agent model — rejected
- **Salt**: Viable but smaller community for network automation use cases — Ansible preferred

---

## 3. Networking

### Decision: pfSense + Traefik + step-ca + NetBird

| Tool | Role |
|---|---|
| pfSense CE + pfBlockerNG | Firewall, DHCP, DNS, VPN, threat blocking |
| Bind 9 | Authoritative internal DNS |
| Traefik v3 | Edge reverse proxy + TLS termination |
| step-ca (Smallstep) | Internal CA for `*.{env}.by-systems.be` |
| WireGuard (pfSense) | Site-to-site + road warrior VPN |
| NetBird | Zero-config mesh VPN for user devices (SSO via Authentik) |
| Cloudflare | Public DNS + DNS-01 ACME challenge |

**Key rules:**
- **Traefik is the only edge proxy.** No Apache/Nginx at the network edge. Internal service reverse proxies (e.g., GitLab's built-in Nginx) run on localhost only (`listen_port 8080`, `listen_https false`).
- **Internal TLD is `.internal`** — avoids mDNS `.local` conflicts per RFC 6762.
- **Split DNS:** `*.{env}.by-systems.be` → Pi-hole/OPNsense Unbound (internal); `*.{env}.by-systems.be` → Cloudflare DNS-01 (external).
- **Dual-stack everywhere:** IPv4 + IPv6, A + AAAA DNS records.

**Rationale:**
- pfSense provides a single, auditable firewall/gateway — mature, widely deployed in SME/telecom environments
- pfBlockerNG replaces Pi-Hole for DNS-level threat blocking — consolidated into the firewall
- Traefik centralises TLS management and integrates natively with Docker and Kubernetes; no Nginx Proxy Manager needed
- step-ca provides a proper internal CA — avoids self-signed cert sprawl
- NetBird simplifies zero-trust mesh VPN for developer devices without complex Tailscale infrastructure

**Alternatives considered:**
- **Nginx Proxy Manager**: Replaced by Traefik — better K8S/Docker native integration, API-driven config
- **Pi-Hole**: Replaced by pfBlockerNG — consolidates DNS threat blocking into existing firewall
- **Headscale (self-hosted Tailscale)**: Viable alternative to NetBird — pending final decision, functionally equivalent
- **mDNS `.local`**: Rejected for service FQDNs — conflicts with RFC 6762; mDNS (Avahi) kept only for IoT/printer discovery

---

## 4. CI/CD & Git

### Decision: GitLab CE (self-hosted) as primary, GitHub as secondary

| Tool | Role |
|---|---|
| GitLab CE | Primary Git + CI/CD + issues + all registries |
| GitLab Runner | CI execution (unlimited self-hosted) |
| GitLab Container Registry | Container image storage — OCI, **no Docker Hub** |
| GitLab Package Registry | **Internal package publishing only** — npm, NuGet, PyPI, Maven, Helm, Cargo, Conan, Terraform. Not a proxy. |
| Kaniko | Container image builder (rootless, daemonless) — **replaces `docker build`** |
| **Nexus Repository OSS** | **Universal proxy + cache** — npm, NuGet, Go (GOPROXY), PyPI, Conan, Maven, Docker, Helm. Replaces Verdaccio + Athens. |
| GitHub | Public/open source mirrors only |
| Trivy | Container, IaC, and secret scan + SBOM |
| Gitleaks | Git secret scanning (pre-commit + pipeline) |
| OWASP Dependency-Check | Application dependency CVE scan |
| Commitizen | Conventional commits enforcement |
| release-please | Semantic versioning + changelog generation |

**Rationale:**
- GitLab CE is a complete DevOps platform — Git, CI/CD, registry, issues in one self-hosted instance
- **GitLab Package Registry** handles internal package publishing natively via `CI_JOB_TOKEN` — no extra config. Covers Go, C#, C++, JS, Python, and more.
- **GitLab Package Registry does NOT proxy upstream registries** — it is publish-only for internal packages
- **Nexus OSS** fills the proxy/cache gap as a single language-agnostic layer in front of all public registries (npmjs.org, nuget.org, pkg.go.dev, pypi.org, conan.io, etc.)
- **Resilience principle:** any package fetched once through Nexus is cached permanently — upstream removal, deprecation, or outage has no impact on builds. Packages cannot disappear from your build pipeline.
- **Kaniko** builds container images inside GitLab Runner without a Docker daemon — rootless, unprivileged. Dockerfile `RUN` steps resolve dependencies via Nexus, not the public internet.
- Nexus replaces Verdaccio (npm-only) and Athens (Go-only) with a single service — reduces operational overhead
- GitHub used for public/open-source repos; GitLab remains source of truth for internal work
- Trivy + Gitleaks + OWASP DC = layered scanning: container layers, git history, application dependencies

**Git signing policy:** Ed25519 GPG keys, Vault-backed, annual rotation.

**Kaniko secret handling policy:**

| Rule | Detail |
|---|---|
| Registry auth | Automatic via `CI_JOB_TOKEN` → `/kaniko/.docker/config.json`. Never passed as CLI argument. |
| `--build-arg` for secrets | **Forbidden.** Values appear in `ps aux` and are stored in image layer history. |
| Build-time secrets | Use `RUN --mount=type=secret` (BuildKit syntax). Kaniko mounts as tmpfs — not stored in any layer. |
| Runtime secrets | Injected at container start via Vault Agent or K8S secret. Never baked into image. |
| Post-build validation | `trivy image` runs after every Kaniko build to verify no secrets leaked into layers. |

**Alternatives considered:**
- **Harbor**: Container images + Helm only — no npm/NuGet/PyPI/Go support. GitLab built-in registry covers container storage. Harbor remains optional for multi-tenant registry with image replication.
- **`docker build` in privileged Runner**: Rejected — requires privileged mode, Docker daemon, security risk.
- **Buildah / BuildKit standalone**: Viable alternatives to Kaniko; Kaniko preferred for GitLab Runner simplicity.
- **Verdaccio**: npm/pnpm proxy only — replaced by Nexus OSS which covers all formats in one service.
- **Athens**: Go module proxy only — replaced by Nexus OSS Go proxy support.
- **Artifactory OSS**: Maven/Gradle only in free tier — too limited. Nexus OSS covers all required formats free.
- **Gitea/Forgejo**: Lighter but lacks native CI/CD — GitLab CE preferred for integrated DevOps.
- **GitHub Actions self-hosted**: Viable but requires GitHub as primary — GitLab CE self-hosted gives full control.

---

## 5. Identity & Access

### Decision: Authentik + HashiCorp Vault + Vaultwarden + Teleport CE + Apache Guacamole

| Tool | Role |
|---|---|
| Authentik | SSO provider (OIDC/SAML) |
| HashiCorp Vault (OSS) | Machine secrets (CI/CD, Ansible, services, K8S) |
| Vaultwarden | Human credentials (self-hosted Bitwarden-compatible) |
| **Teleport CE** | **Primary bastion**: certificate SSH, K8S access, DB access, session recording, audit trail, MFA, OIDC |
| Apache Guacamole | Secondary OOB gateway: browser RDP/VNC for Windows VMs and non-technical users |

**Access flow:** External → WireGuard/NetBird → pfSense → Traefik → Authentik SSO → Service

**Bastion access flow:**
- Engineers/DevOps → **Teleport CE** → certificate-based SSH (no shared keys, no `authorized_keys`) + K8S `kubectl` + DB
- Windows/VNC/non-technical → **Guacamole** → browser RDP/VNC behind Authentik SSO
- Direct SSH to production blocked at pfSense — all access flows through Teleport

**Rationale:**
- Authentik provides OIDC/SAML SSO with self-hosted control and integrates with NetBird, Teleport, Guacamole, and all platform services
- Vault separates machine secrets from human credentials — CI/CD and Ansible never consume credentials from a password manager
- Vaultwarden replaces cloud-based password managers — all credentials stay on-premise
- **Teleport CE** is purpose-built for DevOps bastion access — session recording and audit trail are required for ISO 27001 / NIS2 compliance; certificate-based SSH eliminates shared key management
- Guacamole covers the Windows RDP/VNC gap that Teleport handles less elegantly — complementary, not redundant

**Compliance note:** Teleport session recording satisfies ISO 27001 A.9 (access control) and NIS2 audit requirements. All privileged access is logged, recorded, and replayable.

**Alternatives considered:**
- **Guacamole as primary bastion**: No session recording, no certificate SSH, no K8S/DB access — insufficient for ISO 27001/NIS2 audit. Kept as secondary for Windows/VNC only.
- **Boundary OSS (HashiCorp)**: Strong Vault integration but lacks built-in session recording in OSS tier — Teleport preferred.
- **Keycloak**: Mature SSO alternative, heavier footprint than Authentik — Authentik chosen for simpler UX and smaller resource footprint
- **1Password / Bitwarden cloud**: Cloud-hosted — rejected for on-premise compliance posture
- **HashiCorp Vault Enterprise**: Adds cost — OSS version sufficient for PoC scope

---

## 6. Storage & Databases

### Decision: PostgreSQL (Patroni) + Redis (Sentinel) + MinIO + Neo4J Community

| Tool | Role |
|---|---|
| PostgreSQL (Patroni) | Primary relational DB cluster (GitLab, Vault, NetBox, Authentik, Zabbix) |
| Redis (Sentinel) | Cache, sessions, queues |
| MinIO | S3-compatible object storage (Loki backends, backups, CI artifacts) |
| Neo4J Community | Graph CMDB (dependency/relationship mapping, AI-queryable) |

**Strategy:** Start single-instance → cluster when load justifies it (Patroni for PostgreSQL HA, Redis Sentinel for Redis HA).

**Rationale:**
- PostgreSQL is the standard OSS relational database — all major platform tools (GitLab, Vault, NetBox, Authentik) support it natively
- Patroni provides HA clustering without proprietary extensions
- MinIO provides S3-compatible object storage on-premise — eliminates dependency on AWS S3 for Loki log storage and pipeline artifact storage
- Neo4J Community adds a graph layer for CMDB dependency mapping and AI agent queries — uniquely suited for relationship-heavy telecom/broadcast asset graphs

**Flow:** NetBox (structured source of truth) → sync → Neo4J (graph CMDB) → AI agent queries

**Alternatives considered:**
- **MySQL/MariaDB**: Less preferred for platform services that express first-class PostgreSQL support — rejected
- **CockroachDB**: Distributed SQL, complex operational overhead for PoC scale — rejected
- **AWS S3 / Cloudflare R2**: External dependency for object storage — MinIO self-hosted preferred for air-gap compatibility

---

## 7. IPAM / CMDB

### Decision: NetBox + Neo4J

| Tool | Role |
|---|---|
| NetBox | IPAM + DCIM source of truth (devices, VLANs, IPv4/IPv6, circuits, VRFs) |
| Neo4J Community | Graph CMDB — dependency and relationship mapping |

**Rationale:**
- NetBox is the industry standard for network/infrastructure asset management — supports Ansible dynamic inventory via `netbox.netbox` collection
- Neo4J adds graph semantics that NetBox's relational model cannot express well — dependency chains, impact analysis, RCA traversal
- AI agent observability queries traverse Neo4J for downstream impact assessment

**Alternatives considered:**
- **NetBox alone**: Sufficient for IPAM but lacks graph query capability for complex dependency analysis
- **i-doit**: CMDB alternative, heavier proprietary footprint — NetBox + Neo4J preferred for OSS purity

---

## 8. Monitoring & Observability

### Decision: Prometheus + Grafana + Loki + Zabbix

| Tool | Role |
|---|---|
| Prometheus + Alertmanager | Metrics collection + alert routing |
| Grafana | Dashboards + observability URL registry |
| Loki | Log aggregation (Syslog RFC5424, S3/MinIO backend) |
| Zabbix | SNMP/agentd monitoring for network devices and legacy hosts |

**Observability URL registry pattern:**
- Every service has standard Grafana dashboard sets tagged `service:{name}`, `env:{env}`, `scope:{scope}`
- No hardcoded dashboard URLs — Grafana API is the live registry
- NetBox custom field `prometheus_job` links each service record to its metrics
- AI agent observability flow: NetBox → Prometheus → Grafana → Loki → Neo4J (impact) → synthesised response with direct links

**Rationale:**
- Prometheus + Grafana + Loki is the de-facto OSS observability stack (PLG stack)
- Zabbix retained for SNMP polling of network devices and legacy hosts not covered by Prometheus exporters
- Icinga2 is referenced from the CLIENT-A audit but is optional — Zabbix covers the same use case

**Alternatives considered:**
- **Elasticsearch/ELK**: Heavier resource footprint than Loki for log aggregation — rejected for PoC scale
- **Datadog / New Relic**: Proprietary, SaaS — rejected on cost and data sovereignty grounds
- **InfluxDB + Telegraf**: Alternative metrics stack — Prometheus preferred for broader exporter ecosystem

---

## 9. Security & Compliance

### Decision: Layered security stack targeting ISO 27001, NIS2, DORA, GDPR

**Scan layers (left to right in the pipeline):**

| Layer | Tool |
|---|---|
| Code (SAST) | Semgrep |
| Dependencies (CVE) | OWASP Dependency-Check |
| IaC security | Checkov |
| Containers (scan + SBOM) | Trivy |
| Git secrets (pre-commit + CI) | Gitleaks |
| Network vulnerabilities | OpenVAS / Greenbone |
| Runtime K8S/container behaviour | Falco |
| Host hardening | Lynis |
| Compliance scanning | OpenSCAP (CIS, STIG, ISO profiles) |
| SIEM + IDS/IPS | Wazuh |
| K8S security posture | Prowler |
| Vulnerability aggregation hub | DefectDojo |
| GRC (risk, compliance, audit) | CISO Assistant (intuitem) |

**Rationale:**
- Each layer catches distinct vulnerability classes — no single tool covers the full surface
- DefectDojo aggregates all scanner outputs into a single vulnerability management dashboard — reduces tool-switching
- CISO Assistant provides the GRC governance layer (risk register, compliance evidence, audit trails) across ISO 27001, NIS2, NIST, SOC2, DORA, GDPR
- Wazuh combines SIEM + IDS/IPS + compliance dashboards (ISO 27001, NIS1/NIS2) in a single OSS agent/server model
- Falco provides runtime syscall-level detection for K8S workloads — catches threats that static scanning misses

**Alternatives considered:**
- **Commercial SIEM (Splunk, QRadar)**: High licensing cost — Wazuh covers the use case for OSS
- **Snyk**: SaaS model for dependency/container scanning — OWASP DC + Trivy provide equivalent coverage self-hosted
- **Eramba**: Alternative GRC platform — CISO Assistant preferred for broader framework coverage (100+ frameworks)

---

## 10. Documentation

### Decision: Markdown + PlantUML + KROKI + Sphinx

| Tool | Role |
|---|---|
| Markdown + PlantUML | Source format |
| KROKI | Self-hosted diagram renderer |
| Sphinx | Build engine (→ PDF, HTML, DOCX) |
| Draw.io | Architecture diagrams |
| ADR (Markdown, `adr/` folder) | Architecture Decision Records |

**Rationale:**
- Docs-as-code: all documentation lives in Git, versioned alongside the infrastructure it describes
- KROKI renders PlantUML, Mermaid, D3, BPMN, and Graphviz from a single self-hosted container — no external renderer dependency
- Sphinx provides professional multi-format output (PDF, HTML, DOCX) for customer-facing deliverables

**Alternatives considered:**
- **MkDocs**: Simpler than Sphinx but weaker multi-format output — Sphinx preferred (final decision pending)
- **Notion / Confluence**: SaaS, external dependency — rejected; docs belong in the repo
- **GitBook**: SaaS — rejected on same grounds

---

## 11. Naming Convention Summary

Full details in `docs/naming-convention.md`. Key rules:

| Domain | Pattern | Example |
|---|---|---|
| Repository | `{scope}-{component}-{qualifier}` | `platform-gitlab-core` |
| Branch | `{type}/{issue-id}-{short-description}` | `feat/42-add-vault-integration` |
| Commit | `{type}({scope}): {description}` | `feat(platform): add vault compose stack` |
| FQDN | `{service}.{env}.by-systems.be` | `vault.poc.by-systems.be` |
| Device | `{function}-{vendor}-{env}-{number}` | `sw-arista-prod-01` |
| VM | `vm-{service}-{env}-{number}` | `vm-gitlab-prod-01` |
| K8S namespace | `{scope}-{env}` | `platform-prod` |
| Container image | `{registry}/{scope}/{component}:{version}-{env}` | `registry.poc.by-systems.be/platform/gitlab-core:1.2.3-prod` |

---

## Consequences

### Positive

- **Full OSS stack:** No licensing cost, no vendor lock-in. All tools have active communities and self-hosting support.
- **Integrated security posture:** Defence-in-depth across code, container, runtime, and compliance layers from day one.
- **Single-pane CMDB:** NetBox + Neo4J provides both structured IPAM and graph-based dependency analysis — supports AI agent observability queries.
- **Customer-ready reference:** The two-tier model (base + optional modules) maps directly to customer engagement profiles (IT-only vs broadcast/VoIP/CCTV).
- **Docs-as-code:** All documentation versioned in Git, diagrams rendered self-hosted — no external SaaS dependency.
- **Kubernetes migration path:** All Tier 1 tools have Docker and Helm K8S deployment options — Docker first, K8S when needed.

### Negative / Trade-offs

- **Operational complexity:** Running this full stack requires a competent DevOps operator. Not suitable for unattended SME deployment without a managed service wrapper.
- **Storage and compute:** The full stack (GitLab, Vault, Wazuh, DefectDojo, CISO Assistant, Neo4J, monitoring stack, etc.) has a significant resource footprint — validated Proxmox hypervisor sizing is required before production deployment.
- **PostgreSQL single instance at start:** Patroni HA cluster is deferred until load justifies it — initial PoC runs on single-instance PostgreSQL, which is a risk for platform services during failure.
- **Pending minor decisions:**
  - Sphinx vs MkDocs (Sphinx preferred, not yet final)
  - Headscale vs NetBird for mesh VPN (both viable; NetBird currently selected)
  - Neo4J D3.js visualisation layer vs alternative graph UI

### Risks

| Risk | Mitigation |
|---|---|
| GitLab CE capacity (GitLab is resource-heavy) | Dedicate a VM with ≥8 GB RAM; Patroni DB off-host |
| Wazuh agent sprawl (noisy alerts at PoC scale) | Tune alert thresholds early; suppress dev/staging noise |
| Neo4J sync lag from NetBox | Implement event-driven sync (NetBox webhooks → sync job) rather than polling |
| step-ca certificate rotation | Automate renewal via ACME — test before production cutover |

---

## References

- `docs/stack.md` — full tool inventory with license and deployment options
- `docs/naming-convention.md` — complete naming rules across all domains
- `docs/archive/brainstorm-2026-03-25.md` — session notes and locked decisions (archived)
- CLIENT-A CMDB audit (2024) — prior art reference architecture
