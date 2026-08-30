# BY-SYSTEMS Platform — Technology Stack
**Last updated:** 2026-04-01
**Status:** Locked (PoC phase)
**Philosophy:** Open source, self-hosted, free tier — no vendor lock-in

All tools run on Docker (minimum) with Kubernetes migration path via Helm.

---

## Tier 1 — Base Platform (all deployments)

### Compute & Virtualisation

| Tool | Role | License | Free Plan | Docker | K8S |
|---|---|---|---|---|---|
| Proxmox VE | Hypervisor (VM + LXC) | AGPL v3 | ✅ Community | — | — |
| k3s | Lightweight Kubernetes | Apache 2.0 | ✅ | ✅ | ✅ Helm |
| k9s | K8S terminal UI | Apache 2.0 | ✅ | — | — |
| Headlamp | K8S web UI | Apache 2.0 | ✅ | ✅ | ✅ Helm |
| Packer | Golden VM image builder (bare metal / Proxmox) | BUSL 1.1 ⚠️ | ✅ | ✅ | — |

---

### Infrastructure as Code

| Tool | Role | License | Free Plan | Docker | K8S |
|---|---|---|---|---|---|
| Terraform (Proxmox provider) | VM/LXC provisioning | BUSL 1.1 ⚠️ | ✅ | ✅ | — |
| Ansible | Config management | GPL v3 | ✅ | ✅ | — |
| arista.eos (collection) | Arista switch automation | Apache 2.0 | ✅ | — | — |
| netbox.netbox (collection) | NetBox dynamic inventory | Apache 2.0 | ✅ | — | — |

---

### Networking

| Tool | Role | License | Free Plan | Docker | K8S |
|---|---|---|---|---|---|
| pfSense CE + pfBlockerNG | Firewall, DHCP, DNS, VPN, threat blocking | Apache 2.0 | ✅ | — | — |
| Bind 9 | Authoritative internal DNS | MPL 2.0 | ✅ | ✅ | — |
| WireGuard | Site-to-site + road warrior VPN | GPLv2 | ✅ | ✅ | — |
| NetBird | Zero-config mesh VPN (SSO OIDC) | BSD-3 | ✅ Self-hosted | ✅ | ✅ |
| Traefik v3 | Reverse proxy + TLS (everywhere) | MIT | ✅ | ✅ | ✅ Helm |
| step-ca (Smallstep) | Internal CA (*.{env}.by-systems.be) | Apache 2.0 | ✅ | ✅ | ✅ Helm |
| Cloudflare | Public DNS + DNS-01 ACME | Proprietary | ✅ Free tier | — | — |

> **Rule:** Traefik is the only edge proxy. No Apache/Nginx at edge. Internal services (e.g. GitLab Nginx) are kept on localhost port only.

> **FQDN pattern:** `{service}.{env}.by-systems.be` (e.g. `vault.poc.by-systems.be`) — split DNS, Pi-hole/OPNsense Unbound internal + Cloudflare external.

---

### CI/CD & Git

| Tool | Role | License | Free Plan | Docker | K8S |
|---|---|---|---|---|---|
| GitLab CE (self-hosted) | Git + CI/CD + issues + all registries | MIT | ✅ | ✅ | ✅ Helm |
| GitLab Runner | CI execution (unlimited) | MIT | ✅ | ✅ | ✅ |
| GitLab Container Registry | Container image storage (OCI) — **no Docker Hub** | MIT | ✅ (built-in) | — | — |
| GitLab Package Registry | Internal package publishing: npm, NuGet, PyPI, Maven, Helm, Cargo, Conan, Terraform — **publish only, no proxy** | MIT | ✅ (built-in) | — | — |
| Kaniko | **Container image builder** (rootless, daemonless, runs in CI) — replaces `docker build` | Apache 2.0 | ✅ | ✅ | ✅ |
| **Nexus Repository OSS** | **Universal proxy + cache** for all public registries: npm, NuGet, Go (GOPROXY), PyPI, Conan, Maven, Docker, Helm — language-agnostic, replaces Verdaccio + Athens | Apache 2.0 | ✅ Self-hosted | ✅ | ✅ Helm |
| GitHub | Public/open source repos mirror only | Proprietary | ✅ Free tier | — | — |
| Commitizen | Conventional commits | MIT | ✅ | ✅ | — |
| release-please | Semantic versioning + changelog | Apache 2.0 | ✅ | — | — |
| Trivy | Container/IaC/secret scan + SBOM | Apache 2.0 | ✅ | ✅ | ✅ |
| Gitleaks | Git secret scanning | MIT | ✅ | ✅ | — |
| OWASP Dependency-Check | App dependency vulnerability scan | Apache 2.0 | ✅ | ✅ | — |

> **Package layer architecture — two complementary roles, no overlap:**
>
> | Layer | Tool | Role |
> |---|---|---|
> | Proxy + cache | **Nexus OSS** | Sits in front of all public registries. First pull caches permanently — if upstream removes a package, you still have it. Covers: npm, NuGet, Go, PyPI, Conan, Maven, Docker, Helm |
> | Private publish | **GitLab Package Registry** | Authoritative store for internal libs. CI/CD native via `CI_JOB_TOKEN`. Not a proxy. |
> | Image build | **Kaniko** | Builds container images in CI (rootless). Dockerfile `RUN` steps hit Nexus, not the internet. |
> | Image storage | **GitLab Container Registry** | Stores and serves built images. |
>
> **Flow:** `Developer / CI → Nexus (cache hit or fetch+store from public registry) → upstream only if not cached`
>
> **Resilience:** any package fetched once is stored in Nexus forever. Upstream deprecation, removal, or outage has no impact on builds.

> **Kaniko secret handling rules:**
> - Registry auth: automatic via `CI_JOB_TOKEN` → `/kaniko/.docker/config.json` (never on CLI)
> - **Never use `--build-arg` for secrets** — values appear in `ps aux` and in image layer history
> - Build-time secrets (e.g. private feed token): use `RUN --mount=type=secret` → Kaniko mounts as tmpfs, not stored in any layer
> - Runtime secrets: injected at container start via Vault Agent or K8S secret — never baked into the image
> - Post-build: `trivy image` runs after every Kaniko build to verify no secrets leaked into layers

> **Git signing:** Ed25519 GPG keys, Vault-backed, annual rotation

---

### Identity & Access

| Tool | Role | License | Free Plan | Docker | K8S |
|---|---|---|---|---|---|
| Authentik | SSO provider (OIDC/SAML) | MIT | ✅ Self-hosted | ✅ | ✅ Helm |
| HashiCorp Vault (OSS) | Machine secrets (CI/CD, Ansible, services) | BUSL 1.1 ⚠️ | ✅ Internal use | ✅ | ✅ Helm |
| Vaultwarden | Human credentials (Bitwarden-compatible) | AGPL v3 | ✅ Self-hosted | ✅ | ✅ |
| **Teleport Community Edition** | **Primary bastion**: certificate-based SSH, K8S `kubectl`, DB access, session recording + replay, full audit trail, MFA, Authentik OIDC | Apache 2.0 | ✅ Self-hosted | ✅ | ✅ Helm |
| Apache Guacamole | **Secondary OOB gateway**: browser-based RDP/VNC for Windows VMs and non-technical users | Apache 2.0 | ✅ | ✅ | ✅ |

> **Access flow:** External → WireGuard/NetBird → pfSense → Traefik → Authentik SSO → Service
>
> **Bastion access flow:**
> - DevOps/engineers: **Teleport CE** → certificate SSH (no shared keys) + K8S + DB + full session recording
> - Windows/VNC/non-technical: **Guacamole** → browser RDP/VNC behind Authentik SSO
> - No direct SSH to production without going through Teleport (enforced by pfSense firewall rules)
> - All sessions recorded and replayable — required for ISO 27001 / NIS2 audit trail

---

### Storage & Databases

| Tool | Role | License | Free Plan | Docker | K8S |
|---|---|---|---|---|---|
| PostgreSQL (Patroni) | Primary relational DB cluster | PostgreSQL / Apache 2.0 | ✅ | ✅ | ✅ Helm |
| Redis (Sentinel) | Cache + sessions + queues | BSD-3 (≤7.2) / RSALv2+SSPL (≥7.4) ⚠️ | ✅ Pin to 7.2 | ✅ | ✅ Helm |
| MinIO | S3-compatible object storage | AGPL v3 | ✅ Self-hosted | ✅ | ✅ Helm |
| Neo4J Community | Graph CMDB (nodes/edges/semantic) | GPL v3 | ✅ Community | ✅ | ✅ |

> **DB strategy:** Start single instance → cluster when load justifies it (Patroni for PG HA, Redis Sentinel for Redis HA)

---

### IPAM / CMDB

| Tool | Role | License | Free Plan | Docker | K8S |
|---|---|---|---|---|---|
| NetBox | IPAM + DCIM source of truth (devices, VLANs, IPv4/IPv6, VRFs, circuits) | Apache 2.0 | ✅ | ✅ | ✅ Helm |
| Neo4J Community | Graph CMDB — dependency/relationship mapping, AI-queryable | GPL v3 | ✅ | ✅ | ✅ |

> **Flow:** NetBox (structured source of truth) → sync → Neo4J (graph CMDB) → AI agent queries

---

### Monitoring & Observability

| Tool | Role | License | Free Plan | Docker | K8S |
|---|---|---|---|---|---|
| Prometheus | Metrics collection | Apache 2.0 | ✅ | ✅ | ✅ Helm |
| Grafana | Dashboards + observability URL registry (API) | AGPL v3 | ✅ Self-hosted | ✅ | ✅ Helm |
| Loki | Log aggregation (Syslog RFC5424, S3 backend) | AGPL v3 | ✅ Self-hosted | ✅ | ✅ Helm |
| Alertmanager | Alert routing + notification | Apache 2.0 | ✅ | ✅ | ✅ Helm |
| Zabbix | SNMP/agentd (network devices, legacy) | GPL v2 | ✅ | ✅ | ✅ |

**Observability URL registry (per service):**
- Every service has a standard Grafana dashboard set: overview, logs, alerts, dependencies
- Grafana dashboard tags: `service:{name}`, `env:{env}`, `scope:{scope}`
- AI agent queries Grafana API: `GET /api/search?tag=service:gitlab&tag=env:prod` → returns dashboard URL dynamically
- NetBox custom field per service: `prometheus_job` (links service record to metrics)
- No hardcoded URLs — Grafana API is the live registry

**AI observability query flow:**
1. NetBox API → service definition + `prometheus_job` label
2. Prometheus API → current metrics (uptime, error rate, latency p95)
3. Grafana API → dashboard URL by service tag
4. Loki API → recent errors for service label
5. Neo4J → downstream impact of any active alerts
6. Response includes: status summary + direct links to dashboard, logs, alerts

---

### Security & Compliance (CISO Stack)

| Tool | Role | License | Free Plan | Docker | K8S |
|---|---|---|---|---|---|
| **Wazuh** | SIEM + IDS/IPS + compliance dashboards (ISO 27001, NIS1/NIS2) | GPL v2 | ✅ Self-hosted | ✅ | ✅ Helm |
| **OpenSCAP** | Automated compliance scanning (CIS Benchmarks, STIG, ISO 27001) | LGPL | ✅ | ✅ | — |
| **Falco** | Runtime K8S/container behavioral security (syscall level) | Apache 2.0 | ✅ | ✅ | ✅ Helm |
| **Lynis** | Host hardening audit (Linux systems) | GPL v3 | ✅ | — | — |
| **Trivy** | Container + IaC + secret scan + SBOM (in CI pipeline) | Apache 2.0 | ✅ | ✅ | ✅ |
| **Gitleaks** | Git secret scanning (pre-commit + CI) | MIT | ✅ | ✅ | — |
| **OWASP Dependency-Check** | Application dependency CVE scan | Apache 2.0 | ✅ | ✅ | — |
| **Checkov** | IaC security scan (Terraform, Ansible, Docker, K8S manifests) | Apache 2.0 | ✅ | ✅ | — |
| **Semgrep** | SAST — static code analysis (OSS rules) | LGPL | ✅ | ✅ | — |
| **Prowler** | K8S + cloud security posture (CIS K8S benchmark, NIS2) | Apache 2.0 | ✅ | ✅ | ✅ |
| **OpenVAS / Greenbone** | Network vulnerability scanner (CVE, unpatched services) | GPL v2 | ✅ Self-hosted | ✅ | — |
| **DefectDojo** | Vulnerability management hub — aggregates all scanner outputs | BSD | ✅ Self-hosted | ✅ | ✅ Helm |
| **CISO Assistant** (intuitem) | GRC platform — Risk, Compliance, Audit, TPRM, Privacy (ISO 27001, NIS2, NIST, SOC2, DORA, GDPR, 100+ frameworks) | AGPL v3 | ✅ Self-hosted | ✅ | ✅ |
| **Eramba** | ISO 27001 program management, risk register, controls | AGPL v3 | ✅ Community | ✅ | — |

> **Standards targeted:** ISO 27001, NIS1, NIS2, DORA, GDPR
> **CISO Assistant** = GRC governance layer (risk register, compliance evidence, audit trails)
> **DefectDojo** = vulnerability aggregation layer (all scanner outputs in one dashboard)
> **Scan layers:** code (Semgrep) → dependencies (OWASP DC) → IaC (Checkov) → containers (Trivy) → network (OpenVAS) → runtime (Falco) → compliance (Wazuh + OpenSCAP) → GRC (CISO Assistant)

---

### Mail & Collaboration (non-prod)

| Tool | Role | License | Free Plan | Docker | K8S |
|---|---|---|---|---|---|
| Mailcow | Self-hosted mail stack: SMTP, IMAP, CalDAV, CardDAV, Webmail (SOGo) | MIT | ✅ Self-hosted | ✅ | — |
| Thunderbird | Mail/calendar client for dev/test/staging validation | MPL 2.0 | ✅ | — | — |

> **Non-prod only.** Mailcow runs against internal `example.com` domain (RFC 2606 reserved) for dev/test/staging/acceptance environments.
> Production mail uses the customer's existing provider (Exchange, Google Workspace, etc.).
> Outbound relay from platform services (GitLab, Grafana, Authentik) uses SMTP relay → customer SMTP in prod.

---

### Documentation & Diagrams

| Tool | Role | License | Free Plan | Docker | K8S |
|---|---|---|---|---|---|
| Markdown + PlantUML | Source format for all docs | MIT / GPL | ✅ | — | — |
| KROKI | Self-hosted diagram renderer (PlantUML, Mermaid, D3, BPMN, Graphviz) | MIT | ✅ Self-hosted | ✅ | ✅ |
| Sphinx | Doc build → PDF, HTML, DOCX | BSD | ✅ | ✅ | — |
| Draw.io | Architecture diagrams | Apache 2.0 | ✅ | ✅ (desktop) | — |
| ADR tools (adr-tools) | ADR management CLI | MIT | ✅ | — | — |
| Git LFS | Binary asset versioning (images, videos, PDFs, exports) in Git repos | MIT | ✅ (GitLab built-in) | — | — |

---

### Screenshot & Media Tooling (workstation)

Standardized tooling for all BY-SYSTEMS workstations. Ensures screenshots are consistent size, annotated, and redacted before sharing.

| Tool | Role | License | Platform |
|---|---|---|---|
| Flameshot | Screenshot capture with annotation + **redaction** (blur/pixelate) | GPL v3 | Linux / Windows / macOS |
| ImageMagick | CLI normalization: resize to standard dimensions, strip EXIF metadata | Apache 2.0 | All (CLI) |
| OBS Studio | Screen recording + video capture | GPL v2 | Linux / Windows / macOS |
| Handbrake | Video compression before LFS commit | GPL v2 | All |

#### Standard screenshot dimensions
```
1920×1080   full screen / terminal
1280×720    dialog / panel
800×600     component / widget detail
```

All screenshots must be:
- exported as `.png` (lossless)
- EXIF-stripped (`ImageMagick mogrify -strip`)
- redacted if they contain credentials, IPs, tokens, or customer data (Flameshot blur tool)
- named following `assets/` naming convention (see `docs/naming-convention.md §15`)

---

### Shared Storage & Collaboration

| Tool | Role | License | Free Plan | Docker | K8S |
|---|---|---|---|---|---|
| Nextcloud | Self-hosted file sync, shared drives, calendar, contacts, collaborative editing | AGPL v3 | ✅ Self-hosted | ✅ | ✅ Helm |
| Nextcloud Desktop | Workstation sync client (Linux / Windows / macOS) | GPL v2 | ✅ | — | — |
| OnlyOffice (Nextcloud app) | Collaborative document editing (Word/Excel/PPT compatible) | AGPL v3 | ✅ Self-hosted | ✅ | — |

> **Relationship to Git LFS:** Git LFS handles versioned binary assets tied to a repo (diagrams, exports, videos referenced in docs). Nextcloud handles unversioned or pre-production shared files (customer documents, raw media, team shared drives). Large finalized assets are committed to LFS; work-in-progress lives in Nextcloud until ready.

> **Repo name:** `platform-nextcloud-core`

---

### Developer Environment

| Tool | Role | License | Free Plan |
|---|---|---|---|
| VSCode | Primary IDE | MIT | ✅ |
| devcontainer | Docker-based dev environment per repo | MIT | ✅ |
| GitLens | Git history + blame | MIT | ✅ |
| Conventional Commits | VSCode Commitizen integration | MIT | ✅ |
| HashiCorp Terraform ext | Terraform syntax + validation | MPL 2.0 | ✅ |
| Ansible (Red Hat) | Ansible syntax + linting | MIT | ✅ |
| Docker ext (Microsoft) | Container management | MIT | ✅ |
| Kubernetes ext (Microsoft) | K8S resource management | MIT | ✅ |
| PlantUML ext | Diagram preview | MIT | ✅ |
| Draw.io ext | Diagram editing | Apache 2.0 | ✅ |
| ShellCheck | Shell script linting | GPL v3 | ✅ |

Language-specific extensions:

| Language | Extensions |
|---|---|
| Go | Go (Google), gopls, dlv |
| C++ | C/C++ (Microsoft), clangd, CMake |
| C# / .NET | C# Dev Kit, .NET install tool |
| Python | Pylance, Black, Ruff |
| TypeScript/JS | ESLint, Prettier, Path IntelliSense |
| YAML | YAML (Red Hat), JSON Schema |
| Shell | ShellCheck, Bash IDE |

---

### Network Hardware

| Device | Automation | Protocol |
|---|---|---|
| Arista 7020 / 7060 / 7048 | Ansible arista.eos + eAPI | REST/gRPC |
| Ubiquiti UniFi APs | UniFi Network Controller (Docker) | HTTP API |
| pfSense | Ansible + pfSense API | REST |

---

### VPS

| Provider | Role |
|---|---|
| Contabo | Staging platform, VPN geo exit, WireGuard OOB relay |

---

## Tier 2 — Optional Modules

### Module: Broadcast (AES67 / SMPTE ST 2110)

| Tool | Role | License | Free Plan |
|---|---|---|---|
| PTP (IEEE 1588) | Hardware timestamps on Arista, ptp4l for SW endpoints | OSS | ✅ |
| ptp4l Prometheus exporter | PTP sync monitoring → Grafana | MIT | ✅ |
| Ansible PTP roles | Custom EOS PTP config automation | — | ✅ |
| IGMP v3 + PIM SM | Source-specific multicast on Arista | — | Built-in |

> **VRF layout:** VRF RED (primary media), VRF BLUE (redundant media — SMPTE 2022-7), VRF MGMT (management with VRF leak)

---

### Module: VoIP (SIP + WebRTC)

| Tool | Role | License | Free Plan | Docker | K8S |
|---|---|---|---|---|---|
| Kamailio | SIP proxy/router (SIP, SIPS, WebRTC) | GPL v2 | ✅ | ✅ | ✅ |
| RTPEngine (cluster) | Media relay + WebRTC ICE/SRTP bridge + call recording (.wav) | GPL v2 | ✅ | ✅ | ✅ |
| Coturn | STUN/TURN server (WebRTC NAT traversal) | BSD | ✅ | ✅ | ✅ |
| Homer (HEP) | SIP/VoIP capture + analysis + replay | MIT | ✅ | ✅ | ✅ |
| Redis (shared) | RTPEngine cluster state (node registry, call state) | BSD | ✅ (shared) | ✅ | ✅ |
| MinIO (shared) | Recording storage (.wav files — per-channel or mixed) | AGPL v3 | ✅ (shared) | ✅ | ✅ |

**Kamailio access control:**
- Registration: `auth_db` + `usrloc` + `registrar` modules → credentials in PostgreSQL
- Whitelist/Blacklist: `permissions` module → `address` table (IP/CIDR, allow/deny)
- Domain whitelist/blacklist: `domain` module
- SIP metadata filtering: any header field (From, To, Contact, User-Agent, Via, P-Asserted-Identity, custom X-* headers, Call-ID patterns)
- MAC address: `X-Device-MAC` header + AVP matching (Layer 2 only — on-premise phones)
- External threat feeds: pfBlockerNG IP lists / Fail2ban → sync → Kamailio `htable` or `address` table
- Decision flow: src IP → domain → SIP headers → credentials → accept/403/407

**SIP transaction monitoring:**
- Homer (HEP): full SIP transaction tree per Call-ID (INVITE → 1xx → 200 → ACK → BYE)
- RTPEngine per-call media stats: packets, jitter, loss via `rtpengine_query()`
- Prometheus exporters → Grafana dashboards per call/trunk/user

**SDP / media processing:**
- `sdpops` module: parse/modify SDP in routing script (codec filter, bandwidth, video detection)
- RTPEngine transcoding: Opus↔G.711/G.729 (audio), VP8/VP9↔H.264 (video)
- Codec enforcement: allow/deny list per trunk/user

**Media bridging matrix (WebRTC ↔ SIP):**
- WebRTC (DTLS-SRTP + ICE + Opus/VP8) ↔ SIP (RTP/SRTP + G.711 + H.264) via RTPEngine
- Audio: Opus ↔ G.711/G.729 (transcoded)
- Video: VP8/VP9 ↔ H.264 (transcoded or dropped)
- Messages: SIP MESSAGE ↔ WebRTC data channel (custom Kamailio handler)
- Bidirectional: WebRTC→SIP and SIP→WebRTC

**Virtual SIP Codec (svc-virtual-sip-codec):**
- Dockerized SIP UA — registers to Kamailio as normal SIP endpoint
- Can call or be called (SIP↔SIP, WebRTC↔SIP via RTPEngine bridge)
- Routes audio to/from physical AES67 sound card via ALSA/JACK
- Stack: Baresip (PoC/config-driven) → PJSUA2 Python (production/custom logic)
- REST API: register, call, hangup, route-audio, status
- Docker: `--device /dev/snd` passthrough for AES67 card
- K8S: device plugin or hostPath for `/dev/snd`, `SYS_TIME` cap for PTP
- Multiple instances = multiple virtual codec channels (one pod per channel)
- Repo: `svc-virtual-sip-codec`

**Management UI (custom — Siremis replaced):**
- Backend: Go or Python FastAPI wrapping Kamailio JSONRPC + PostgreSQL
- Frontend: React/Vue SPA
- Features: subscriber mgmt, whitelist/blacklist, RTPEngine cluster node control, live call monitoring, SIP transaction timeline, recording browser (.wav from MinIO), media stats dashboards
- Repo: `platform-voip-ui`

**RTPEngine cluster design:**
- Multiple RTPEngine nodes, Kamailio load-balances via `rtpengine_load_manage()`
- Nodes can be dynamically enabled/disabled per Kamailio (zero-downtime maintenance)
- Redis for cluster state sharing between nodes
- Recording: per-channel (separate .wav — caller/callee independent tracks) or mixed mono
- Recording output → MinIO S3 bucket via RTPEngine `--recording-dir` + custom upload hook or direct S3 backend
- TLS: SIPS (SIP over TLS) + SRTP (encrypted media) — mandatory
- WebRTC: ICE negotiation via Coturn, DTLS-SRTP bridging via RTPEngine

---

### Module: CCTV

| Tool | Role | License | Free Plan | Docker |
|---|---|---|---|---|
| Frigate | NVR + AI object detection | MIT | ✅ | ✅ |
| ZoneMinder | Alternative NVR (mature) | GPL v2 | ✅ | ✅ |

---

## Environments

```
dev → test → staging → acceptance → prod
```

Lite track (simple SME): `dev → staging → prod`

---

## IPv4 + IPv6

All services bind dual-stack.
NetBox manages both address families.
DNS: A + AAAA records for every FQDN.
pfSense: IPv4 + IPv6 firewall rules per VLAN.
K8S: dual-stack service/pod CIDR.

---

## Not in scope

| Tool | Reason |
|---|---|
| Pi-Hole | Replaced by pfSense pfBlockerNG |
| Nginx Proxy Manager | Replaced by Traefik |
| Harbor | Container images + Helm only — GitLab built-in registry covers container storage. Optional upgrade for multi-tenant registry with image replication. |
| Verdaccio | npm-only proxy — replaced by Nexus OSS which covers all formats (npm, NuGet, Go, PyPI, Conan, Maven) in one service |
| Athens | Go module proxy only — replaced by Nexus OSS Go proxy support |
| Artifactory OSS | Free tier is Maven/Gradle only — too limited. Nexus OSS covers all required formats free. |
| Rancher | Replaced by k3s + k9s + Headlamp |
| 1Password / Bitwarden cloud | Replaced by self-hosted Vaultwarden |
| Notion / Confluence | Replaced by Markdown + Sphinx in repo |
| Jira (primary) | GitLab Issues is primary; Jira if customer requires |

---

## Tool License Reference

> Source: Opus audit 2026-04-01. See full findings: `workspace/docs/audits/2026-04-01-opus-findings.md`
> Legend: ✅ Yes — no restrictions | ⚠️ Conditional — see rule | ❌ Blocked

| Tool | SPDX License | Type | Owner | License URL | Free Plan Scope | BY-SYSTEMS Client Offering |
|---|---|---|---|---|---|---|
| OPNsense | BSD-2-Clause | OSI OSS | Deciso B.V. | https://github.com/opnsense/core/blob/master/LICENSE | Full — no restrictions | ✅ Yes |
| Pi-hole | EUPL-1.2 | OSI OSS | Pi-hole LLC | https://github.com/pi-hole/pi-hole/blob/master/LICENSE | Full | ✅ Yes |
| Traefik | MIT | OSI OSS | Traefik Labs | https://github.com/traefik/traefik/blob/master/LICENSE.md | Full | ✅ Yes |
| HashiCorp Vault | BUSL-1.1 | Source-available | HashiCorp (IBM) | https://github.com/hashicorp/vault/blob/main/LICENSE | Full for internal/client infra | ⚠️ Cannot resell hosted Vault as a product |
| Vaultwarden | AGPL-3.0 | OSI OSS (copyleft) | Dani Garcia | https://github.com/dani-garcia/vaultwarden/blob/main/LICENSE | Full — includes Premium features | ⚠️ Deploy unmodified = OK. Modify + SaaS = must open-source |
| Authentik | MIT | OSI OSS | Authentik Security Inc. | https://github.com/goauthentik/authentik/blob/main/LICENSE | Full SSO/OIDC/SAML/LDAP | ✅ Yes |
| GitLab CE | MIT | OSI OSS | GitLab Inc. | https://gitlab.com/gitlab-org/gitlab/-/blob/master/LICENSE | Full VCS + CI + registry | ✅ Yes |
| GitLab Runner | MIT | OSI OSS | GitLab Inc. | https://gitlab.com/gitlab-org/gitlab-runner/-/blob/main/LICENSE | Full | ✅ Yes |
| Nextcloud | AGPL-3.0 | OSI OSS (copyleft) | Nextcloud GmbH | https://github.com/nextcloud/server/blob/master/COPYING | Full file sync + apps | ⚠️ Deploy unmodified = OK. Modify + SaaS = must open-source |
| NetBox | Apache-2.0 | OSI OSS | NetBox Labs | https://github.com/netbox-community/netbox/blob/develop/LICENSE | Full IPAM + DCIM | ✅ Yes |
| Nexus OSS | EPL-1.0 | OSI OSS | Sonatype | https://github.com/sonatype/nexus-public/blob/main/LICENSE.txt | All formats (npm, pip, Docker, Maven, Helm) | ✅ Yes |
| Prometheus | Apache-2.0 | OSI OSS | CNCF | https://github.com/prometheus/prometheus/blob/main/LICENSE | Full | ✅ Yes |
| Grafana | AGPL-3.0 | OSI OSS (copyleft) | Grafana Labs | https://github.com/grafana/grafana/blob/main/LICENSE | Full dashboards + alerting | ⚠️ Deploy unmodified = OK. Modify + SaaS = must open-source |
| Loki | AGPL-3.0 | OSI OSS (copyleft) | Grafana Labs | https://github.com/grafana/loki/blob/main/LICENSE | Full log aggregation | ⚠️ Deploy unmodified = OK. Modify + SaaS = must open-source |
| PostgreSQL | PostgreSQL | OSI OSS (permissive) | PostgreSQL Dev Group | https://www.postgresql.org/about/licence/ | Full | ✅ Yes |
| Redis | BSD-3-Clause (≤7.2) / RSALv2+SSPL (≥7.4) | OSS (≤7.2) / Source-available (≥7.4) | Redis Ltd | https://github.com/redis/redis/blob/7.2/LICENSE | Full at ≤7.2 | ✅ Yes at ≤7.2. ❌ Do not use ≥7.4 — pin to `redis:7.2.7-alpine` |
| Unifi Network App | Proprietary | Freeware | Ubiquiti Inc. | https://www.ui.com/eula/ | Free with Ubiquiti hardware | ⚠️ Deploy for clients with Ubiquiti HW = OK. Cannot modify/redistribute |
| Proxmox VE | AGPL-3.0 | OSI OSS (copyleft) | Proxmox Server Solutions | https://git.proxmox.com/?p=pve-manager.git;a=blob;f=COPYING | Full community | ⚠️ Deploy unmodified = OK. Subscription removes nag only |
| step-ca | Apache-2.0 | OSI OSS | Smallstep | https://github.com/smallstep/certificates/blob/master/LICENSE | Full | ✅ Yes |
| Neo4j CE | GPL-3.0 | OSI OSS (copyleft) | Neo4j Inc. | https://github.com/neo4j/neo4j/blob/dev/LICENSE.txt | Full community | ⚠️ Deploy unmodified = OK. Distribute modifications = GPL applies |

### Client Offering Summary

> **Can BY-SYSTEMS build a managed platform for clients on these tools?**
> **YES — with 3 rules:**
> 1. **Pin Redis to `redis:7.2.7-alpine`** (BSD-3). Never upgrade to 7.4+ without license review.
> 2. **Never sell hosted Vault as a product.** Using Vault to manage client infra secrets = OK. Reselling Vault access = BUSL violation.
> 3. **Deploy AGPL tools unmodified** (Grafana, Loki, Vaultwarden, Nextcloud, Proxmox). Unmodified deployment for clients = zero obligation. Modify + offer as SaaS = must open-source modifications.
>
> **Managed service model** (BY-SYSTEMS deploys + operates tools on client infra) = **zero blockers.**
