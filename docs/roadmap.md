# ORG DevOps Platform — Roadmap
**Last updated:** 2026-03-25
**Status:** Active — PoC phase
**Document type:** `roadmap`

> Practical, phased delivery plan for the ORG internal DevOps PoC.
> Each phase must be fully operational before the next begins.
> Naming conventions follow `docs/naming-convention.md`. Stack follows `docs/stack.md`.

---

## Summary

| Phase | Name | Focus | Track |
|---|---|---|---|
| 1 | Foundation | Compute, networking, storage, IaC | Required |
| 2 | Platform Services | GitLab, Nexus OSS, Vault, Authentik, NetBox, Traefik | Required |
| 3 | Observability & Security | Monitoring, SIEM, compliance, GRC | Required |
| 4a | Module: Broadcast | AES67, SMPTE ST 2110, PTP | Optional |
| 4b | Module: VoIP | SIP, WebRTC, Kamailio, RTPEngine | Optional |
| 4c | Module: CCTV | IP cameras, NVR, AI detection | Optional |
| 5 | Customer Delivery | Templates, runbooks, SOW, handoff | Required |

---

## Phase 1 — Foundation

**Goal:** A working, reproducible infrastructure baseline. Nothing runs on a workstation after this phase.

### 1.1 Compute & Hypervisor

- [ ] Deploy Proxmox VE on bare metal (`srv-proxmox-prod-01`)
- [ ] Configure storage: local-lvm for VMs, ZFS/NFS pool for shared storage
- [ ] Validate VM + LXC provisioning (manual first, then IaC)
- [ ] Set up Packer pipeline for golden VM images (Debian/Ubuntu base)

### 1.2 Networking

- [ ] Deploy pfSense CE on dedicated node (`fw-pfsense-prod-01`)
  - Configure: WAN/ISP uplink, VLAN segmentation, DHCP, DNS resolver (Unbound)
  - Enable pfBlockerNG (threat blocking, IP reputation)
  - Configure WireGuard: site-to-site + road warrior profiles
- [ ] Deploy Bind 9 (authoritative internal DNS, `*.org.internal`)
- [ ] Register `org.example` zone on Cloudflare (public DNS + DNS-01 ACME)
- [ ] Configure Arista switches (`sw-arista-prod-01`) via Ansible `arista.eos`
  - VLANs, IGMP snooping, spanning tree, QoS baseline
- [ ] Deploy UniFi Network Controller (Docker) for AP management
- [ ] Configure split DNS: `*.org.internal` → pfSense | `*.org.example` → Cloudflare
- [ ] Deploy NetBird (mesh VPN for user devices, OIDC via Authentik — wire up after Phase 2)

### 1.3 Infrastructure as Code Baseline

- [ ] Init GitLab project structure (bootstrap: manual GitLab instance or temp container)
  - `infra/proxmox-ansible`, `infra/network-arista`, `infra/network-pfsense`
- [ ] Terraform: Proxmox provider wired, first VM provisioned via code
- [ ] Ansible: roles for base hardening (Lynis baseline), package management, NTP, SSH
- [ ] Commit signing: Ed25519 GPG keys configured, enforced in `.gitconfig`
- [ ] Commitizen + pre-commit hooks: installed in all repos

### 1.4 Storage

- [ ] Deploy MinIO (Docker → K8S later) — S3 endpoint for backups and artifacts
- [ ] Deploy PostgreSQL single instance (upgrade to Patroni cluster in Phase 2)
- [ ] Deploy Redis single instance (Sentinel after Phase 2)

**Exit criteria:** VMs provision via Terraform. Network is segmented and firewall-protected. DNS resolves internally and externally. IaC repos exist with CI skeleton. Baseline storage operational.

---

## Phase 2 — Platform Services

**Goal:** Full DevOps platform live. All internal services accessible via SSO, TLS, and Traefik.

### 2.1 Reverse Proxy & TLS

- [ ] Deploy Traefik v3 (K8S Helm or Docker Compose)
  - Integrate with step-ca (Smallstep) for `*.org.internal` TLS
  - Integrate with Cloudflare DNS-01 for `*.org.example` public TLS
  - Enforce HTTPS everywhere; no plain HTTP exposed
- [ ] Deploy step-ca (`platform-traefik-config`, `platform-stepca-config`)
- [ ] Document Traefik/GitLab Nginx passthrough pattern (nginx internal on 8080, Traefik edge)

### 2.2 Identity & Access

- [ ] Deploy Authentik (`platform-authentik-config`)
  - OIDC/SAML provider for all platform services
  - Flows: login, enrollment, MFA (TOTP)
- [ ] Deploy HashiCorp Vault OSS (`platform-vault-config`)
  - Seal/unseal strategy defined (Shamir or auto-unseal)
  - Secret engines: KV (platform secrets), PKI (CA chain backup), SSH (OTP for ops)
  - AppRole + Kubernetes auth methods enabled
- [ ] Deploy Vaultwarden (`platform-vaultwarden-config`) — human credentials, SSO via Authentik
- [ ] Deploy **Teleport CE** (`platform-teleport-core`) — primary bastion
  - Certificate-based SSH for all Linux VMs (replaces `authorized_keys`)
  - K8S `kubectl` access via Teleport proxy
  - DB access (PostgreSQL, MySQL) via Teleport DB proxy
  - Session recording + replay enabled (audit requirement)
  - OIDC via Authentik
  - pfSense rule: block direct SSH port 22 to all VMs except from Teleport proxy
- [ ] Deploy Apache Guacamole (`platform-guacamole-core`) — secondary OOB gateway
  - Browser RDP/VNC for Windows VMs and non-technical users
  - Behind Authentik SSO
- [ ] Wire NetBird OIDC → Authentik

### 2.3 CI/CD — GitLab + Nexus (permanent instance)

- [ ] Deploy GitLab CE (`platform-gitlab-core`) — replaces bootstrap instance
  - PostgreSQL backend (shared cluster)
  - Redis cache (shared instance)
  - MinIO for object storage (artifacts, LFS, registry)
  - Container Registry enabled (built-in)
  - Package Registry enabled (npm, NuGet, PyPI, Maven, Helm, Conan, Cargo, Terraform)
- [ ] GitLab Runners deployed and registered (Docker executor, min. 2)
- [ ] Kaniko integrated for rootless image builds in GitLab CI
- [ ] **Deploy Nexus Repository OSS** (`platform-nexus-core`) — universal proxy + cache layer
  - Proxy repos: npmjs.org, nuget.org, pkg.go.dev (GOPROXY), pypi.org, conan.io, Maven Central, Docker Hub, Helm repos
  - Group repos: one endpoint per format combining proxy + internal (e.g. `npm-group` = npmjs proxy + GitLab internal packages)
  - All workstations and CI pipelines configured to resolve deps via Nexus first
  - Dockerfile `RUN` steps (inside Kaniko) resolve via Nexus — no direct internet access from build
- [ ] Migrate all bootstrap repos into permanent GitLab
- [ ] Configure GitHub mirrors for public repos (one-way push)
- [ ] CI pipeline template (`tpl-pipeline-base`): lint → build (Kaniko) → Trivy scan → Gitleaks → test → deploy
- [ ] Gitleaks pre-commit hook + CI stage active in all repos
- [ ] release-please configured on platform repos

### 2.4 IPAM & CMDB

- [ ] Deploy NetBox (`platform-netbox-config`)
  - Model all devices, VMs, VLANs, IPv4/IPv6 prefixes, VRFs, circuits
  - Custom fields per service: `repo_url`, `prometheus_job`, `grafana_tag`, `version`, `lifecycle`
  - Populate NetBox Ansible inventory plugin (`netbox.netbox`)
- [ ] Deploy Neo4J Community (`platform-neo4j-cmdb`)
  - Initial graph: nodes (services, VMs, switches, VLANs) + edges (depends-on, hosted-on, connected-to)
  - Sync pipeline: NetBox → Neo4J (Ansible or Python script, scheduled)

### 2.5 Kubernetes

- [ ] Deploy k3s cluster (1 control-plane + 2 workers minimum)
  - Dual-stack IPv4/IPv6
  - Namespaces per scope/env: `platform-prod`, `platform-staging`, etc.
- [ ] Deploy k9s + Headlamp (K8S UI)
- [ ] Migrate eligible platform services to Helm charts (Vault, Authentik, Traefik, NetBox, MinIO)
- [ ] Kaniko integrated for rootless image builds in GitLab CI

### 2.6 Documentation Infrastructure

- [ ] Deploy KROKI (`platform-kroki`) — self-hosted diagram renderer
- [ ] Set up Sphinx build pipeline in `doc-runbooks` and `doc-adr`
- [ ] ADR repo initialised: `docs/adr/0001-platform-stack-decisions.md` (all tool choices)
- [ ] VSCode devcontainer templates published in `tpl-repo-infra` and `tpl-repo-app`

**Exit criteria:** All platform services accessible at `*.org.internal` via Traefik + SSO. GitLab CI running. NetBox populated. K8S cluster operational. No plaintext secrets in repos.

---

## Phase 3 — Observability & Security

**Goal:** Full-stack observability, automated security scanning, compliance dashboards live.

### 3.1 Monitoring & Observability

- [ ] Deploy Prometheus + Alertmanager (`platform-monitoring-stack`)
  - Scrape targets: all platform services, K8S cluster, Proxmox, pfSense, Arista (SNMP), NetBox
  - Every service has `prometheus_job` set in NetBox
- [ ] Deploy Grafana — dashboards per service (overview, logs, alerts, dependencies)
  - Dashboard tags: `service:{name}`, `env:{env}`, `scope:{scope}`
  - AI observability query flow: NetBox API → Prometheus API → Grafana API → Loki API → Neo4J
- [ ] Deploy Loki (Syslog RFC5424 ingestion, MinIO S3 backend)
  - Promtail / syslog-ng forwarder on all VMs and containers
- [ ] Deploy Zabbix (SNMP/agentd for network devices and legacy services)
- [ ] Alert routing: Alertmanager → Discord / email / PagerDuty (configurable per customer)

### 3.2 Security Scanning Pipeline

- [ ] Trivy: container scan + IaC scan + SBOM + secret detection — active in all CI pipelines
- [ ] Checkov: IaC security scan (Terraform, Ansible, Docker, K8S manifests)
- [ ] Semgrep: SAST — static analysis on app repos (OSS rules)
- [ ] OWASP Dependency-Check: dependency CVE scan on app repos
- [ ] OpenVAS / Greenbone: scheduled network vulnerability scans against all environments
- [ ] Prowler: CIS K8S benchmark + NIS2 posture checks on K8S cluster
- [ ] All scanner outputs feed into **DefectDojo** (`sec-defectdojo`) — single vulnerability dashboard

### 3.3 SIEM & Runtime Security

- [ ] Deploy Wazuh (`sec-wazuh-config`)
  - Agents on all VMs, K8S nodes, Proxmox host
  - Dashboards: ISO 27001, NIS1/NIS2 compliance
  - Alerts → Alertmanager → notification channels
- [ ] Deploy Falco (K8S Helm) — runtime behavioral detection on K8S pods
  - Rules: privilege escalation, unexpected network connections, secret file access
- [ ] OpenSCAP: automated CIS benchmark scans on Linux VMs (scheduled via Ansible)
- [ ] Lynis: host hardening audit run on every new VM (post-Packer, post-Ansible)

### 3.4 Compliance & GRC

- [ ] Deploy CISO Assistant (`sec-ciso-assistant`) — GRC governance
  - Frameworks loaded: ISO 27001, NIS2, DORA, GDPR
  - Risk register, compliance evidence, audit trails
- [ ] Deploy Eramba Community (`sec-eramba`) — ISO 27001 program, controls tracking
- [ ] Link DefectDojo findings → CISO Assistant controls (manual or API bridge)
- [ ] First compliance report generated: ISO 27001 gap analysis

### 3.5 Host Hardening Baseline

- [ ] Lynis hardening score target: ≥ 75 on all production VMs
- [ ] OpenSCAP CIS Level 1 pass on all production VMs
- [ ] SSH: key-only, no root login, Fail2ban active
- [ ] Automatic unattended security updates enabled

**Exit criteria:** Prometheus/Grafana/Loki operational for all services. Wazuh agents deployed everywhere. CI pipelines run full scan suite. DefectDojo aggregating findings. CISO Assistant loaded with target frameworks. Hardening baseline passed.

---

## Phase 4a — Module: Broadcast (optional)

**Trigger:** Customer project requires AES67 / SMPTE ST 2110 / PTP.

- [ ] Arista switches: enable PTP (IEEE 1588 boundary clock, hardware timestamps)
  - Ansible `mod-broadcast-ptp`: EOS PTP config automation
- [ ] VRF layout on Arista: VRF RED (primary media), VRF BLUE (redundant — SMPTE 2022-7), VRF MGMT (leak to RED/BLUE)
- [ ] IGMP v3 snooping + PIM sparse-mode configured per VRF
- [ ] QoS: DSCP marking for media traffic classes
- [ ] ptp4l on software endpoints (Linux VMs): `ptp4l` + `phc2sys` configured, synced to Arista BC
- [ ] Deploy ptp4l Prometheus exporter → Grafana dashboard (`mod-broadcast-ptp`)
- [ ] AES67 endpoint test: register source and destination, validate multicast stream delivery
- [ ] SMPTE 2022-7 hitless switching test (RED/BLUE path)
- [ ] Broadcast monitoring dashboard: PTP offset, IGMP group counts, multicast stream health

**Exit criteria:** PTP lock achieved on all broadcast endpoints. AES67 stream delivered. SMPTE 2022-7 failover validated. Monitoring live.

---

## Phase 4b — Module: VoIP (optional)

**Trigger:** Customer project requires SIP, WebRTC, or unified communications.

- [ ] Deploy Kamailio (`mod-voip-kamailio`)
  - PostgreSQL: subscriber table, address whitelist/blacklist, domain table
  - Modules: `auth_db`, `usrloc`, `registrar`, `permissions`, `domain`, `sdpops`, `rtpengine`
- [ ] Deploy RTPEngine cluster (`mod-voip-rtpengine`)
  - Redis cluster state (shared Redis)
  - Call recording → MinIO S3 (`.wav` per channel or mixed)
  - TLS: SIPS + SRTP mandatory
- [ ] Deploy Coturn (`mod-voip-coturn`) — STUN/TURN for WebRTC NAT traversal
- [ ] Deploy Homer (HEP) (`mod-voip-homer`) — SIP transaction capture and analysis
- [ ] Validate WebRTC ↔ SIP bridging (Opus ↔ G.711/G.729, VP8/VP9 ↔ H.264 via RTPEngine)
- [ ] Deploy `svc-virtual-sip-codec` — dockerised SIP UA → AES67 card bridge (Baresip PoC → PJSUA2)
- [ ] Deploy `platform-voip-ui` — custom management UI (Go/FastAPI backend + React frontend)
  - Subscriber management, whitelist/blacklist, live call monitoring, recording browser
- [ ] SIP threat integration: pfBlockerNG IP feeds → Kamailio `htable` + Fail2ban sync
- [ ] Prometheus exporters for Kamailio + RTPEngine → Grafana VoIP dashboard (calls, jitter, loss)

**Exit criteria:** SIP registration working. WebRTC↔SIP call completes. Recording saved to MinIO. Homer captures full SIP transaction tree. Management UI accessible via SSO.

---

## Phase 4c — Module: CCTV (optional)

**Trigger:** Customer project requires IP camera management or AI-based video analytics.

- [ ] Select NVR: Frigate (AI-first) or ZoneMinder (maturity) — document decision in ADR
- [ ] Deploy chosen NVR (`mod-cctv-frigate` or `mod-cctv-zoneminder`)
- [ ] Configure RTSP/RTP multicast ingestion from IP cameras
- [ ] GPU/NPU passthrough for Frigate AI inference (if available)
- [ ] Object detection rules configured (person, vehicle, etc.)
- [ ] Alert flow: detection event → Alertmanager → notification channel
- [ ] Recordings stored on MinIO or dedicated NAS volume
- [ ] Grafana dashboard: camera uptime, detection events, stream health

**Exit criteria:** Cameras ingested. Motion/object detection events firing. Alerts delivered. Recordings accessible.

---

## Phase 5 — Customer Delivery Readiness

**Goal:** Platform is replicable, documented, and safe to hand off to a customer or an internal team.

### 5.1 Templates & Boilerplate

- [ ] `tpl-repo-infra` — Terraform + Ansible skeleton (Commitizen, pre-commit, CI pipeline, devcontainer)
- [ ] `tpl-repo-app` — App repo skeleton (CI pipeline, SBOM, scan stages, release-please)
- [ ] `tpl-pipeline-base` — Reusable GitLab CI pipeline (lint → build → scan → test → deploy stages)
- [ ] `tpl-discord-setup` — Standard Discord workspace config (channels, webhooks, roles)
- [ ] `.commitlintrc` and `cz.toml` published in `tpl-pipeline-base`

### 5.2 Documentation

- [ ] `docs/adr/` — all major decisions recorded (stack, tool choices, naming)
- [ ] `docs/runbook-*.md` — operational runbooks for: Vault backup/restore, GitLab upgrade, Proxmox snapshot, key rotation
- [ ] `docs/raid-platform-poc.md` — Risks, Assumptions, Issues, Dependencies
- [ ] Sphinx pipeline: docs build to HTML + PDF on `main` merge
- [ ] KROKI diagrams: architecture overview, network topology, CI/CD flow, K8S namespace map

### 5.3 Customer Onboarding Process

- [ ] Customer slug registered in NetBox (tenant slug = `{customer-short-name}`)
- [ ] GitLab subgroup created: `{customer-slug}/infra`, `{customer-slug}/platform`, `{customer-slug}/app`
- [ ] SOW template (`tpl-sow-infra-network.md`) ready for customisation
- [ ] Onboarding checklist: network, compute, DNS, GitLab group, Authentik tenant, Vault namespace
- [ ] `mod-*` decision guide: which optional modules apply per customer profile

### 5.4 Delivery Validation

- [ ] End-to-end smoke test: new VM provisioned via Terraform → Ansible-hardened → registered in NetBox → Prometheus scraping → Grafana dashboard → Wazuh agent active
- [ ] CI pipeline smoke test: commit → build → Trivy scan → Gitleaks → deploy to staging
- [ ] SSO smoke test: login to GitLab, Vault, NetBox, Grafana, Teleport, Guacamole — all via Authentik
- [ ] Bastion test: SSH to a VM via Teleport, verify session recorded and replayable
- [ ] Security posture: DefectDojo shows no Critical/High open findings
- [ ] Compliance: CISO Assistant ISO 27001 coverage ≥ 60%

**Exit criteria:** Another engineer can stand up the full platform from templates + runbooks without tribal knowledge. Customer SOW can be generated in < 1 day. PoC is demonstrable end-to-end.

---

## Dependencies & Sequencing Notes

```
Phase 1 (Foundation)
  └── Phase 2 (Platform Services)          ← requires Proxmox, pfSense, DNS, storage
        └── Phase 3 (Observability/Sec)    ← requires GitLab CI, Vault, K8S, NetBox
              └── Phase 5 (Delivery)       ← requires all Tier 1 complete
        └── Phase 4a (Broadcast)           ← parallel; requires Arista + Ansible
        └── Phase 4b (VoIP)                ← parallel; requires GitLab, Vault, PostgreSQL, Redis, MinIO
        └── Phase 4c (CCTV)                ← parallel; requires storage, networking
```

- Phases 4a/4b/4c are **independent of each other** and can proceed in parallel once Phase 2 is done.
- Phase 5 runs concurrently with Phase 3 (templates + docs can be built while security stack is deployed).
- PostgreSQL, Redis, MinIO start as single instances in Phase 1 and are clustered (Patroni, Sentinel) during Phase 2 or Phase 3 depending on load.

---

## Pending Decisions (to be resolved before Phase 2 completion)

| Decision | Options | ADR |
|---|---|---|
| Docs build tool | Sphinx (preferred) vs MkDocs (simpler) | `adr/0002-docs-tooling.md` |
| Mesh VPN | NetBird (current) vs Headscale (self-hosted Tailscale) | `adr/0003-mesh-vpn.md` |
| WiFi management | UniFi Controller vs OpenWRT | `adr/0004-wifi-ap.md` |
| Neo4J graph UI | D3.js custom vs Bloom vs neovis.js | `adr/0005-neo4j-ui.md` |
| CCTV NVR | Frigate vs ZoneMinder | `adr/0006-cctv-nvr.md` (Phase 4c) |

---

## Reference Documents

- `docs/stack.md` — full technology inventory
- `docs/naming-convention.md` — naming rules for repos, services, devices, FQDNs
- `docs/archive/brainstorm-2026-03-25.md` — session notes, decisions made, org structure (archived)
- `docs/adr/0001-platform-stack-decisions.md` — _(pending)_ tool choice rationale
