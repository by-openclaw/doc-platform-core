# BY-SYSTEMS DevOps Platform — Brainstorm Session
**Date:** 2026-03-25  
**Status:** In progress — decisions locked, formal docs pending

---

## Context

BY-SYSTEMS is a **broadcast/telecom system integrator** building an internal DevOps PoC platform.

Goals:
- Replace manual/legacy workflows with proper DevOps tooling
- Serve as reference architecture for SME customers
- Support broadcast/telecom integration projects
- All tooling: open source, self-hosted, free tier where possible

Reference work: RTBF CMDB audit (2024) — 62-slide PowerPoint covering production app landscape, CI/CD pipeline, infrastructure, monitoring, CMDB with Neo4J.

---

## Framework Structure — Two Tiers

### Tier 1 — Base platform (all deployments)
Standard DevOps stack, IT-focused, required for all customers.

### Tier 2 — Optional modules (per customer profile)
- `module-broadcast` — AES67, SMPTE ST 2110, PTP, VRF RED/BLUE
- `module-voip` — SIP, WebRTC, Kamailio, Janus, Homer
- `module-cctv` — IP cameras, RTSP/RTP multicast, Frigate/ZoneMinder

---

## Full Stack — Locked Decisions

### Compute
| Tool | Role |
|---|---|
| Proxmox | Hypervisor (VM + LXC) — replaces VMware |
| k3s | Lightweight Kubernetes |
| kubectl + k9s | K8S daily ops |
| Headlamp | K8S web UI (OSS) |
| Kaniko | Rootless container image builds in CI |
| Packer | Golden image builds (replaces VMware OVA workflow) |

### Infrastructure as Code
| Tool | Role |
|---|---|
| Terraform (Proxmox provider) | VM/LXC provisioning |
| Ansible | Configuration management, network automation |
| Ansible collections: arista.eos, netbox.netbox | Switch config + NetBox dynamic inventory |

### VPS / Cloud
| Tool | Role |
|---|---|
| Contabo VPS | Public-facing services, staging environment, VPN geo exit, WireGuard OOB relay |

### Networking
| Tool | Role |
|---|---|
| pfSense + pfBlockerNG | Firewall, NAT, VLAN, DHCP, DNS resolver, VPN, threat blocking |
| Bind | Authoritative internal DNS |
| Cloudflare | Public DNS + DNS-01 ACME challenge |
| WireGuard (pfSense) | Site-to-site + road warrior VPN |
| NetBird | Zero-config mesh VPN for user devices (SSO via Authentik) |
| Traefik | Reverse proxy + TLS everywhere (no Nginx/Apache at edge) |
| step-ca (Smallstep) | Internal CA for *.by-systems.internal |
| Internal TLD | `.internal` (avoids mDNS .local conflict — RFC 6762) |
| Public TLD | `by-systems.be` (Cloudflare) |
| Split DNS | by-systems.internal (pfSense Unbound) + by-systems.be (Cloudflare) |
| mDNS | Avahi — IoT/printer discovery only, not for services |
| IPv4 + IPv6 | Dual-stack everywhere |

### Network Hardware
| Device | Management |
|---|---|
| Arista 7020/7060/7048 | Ansible arista.eos + eAPI |
| Ubiquiti UniFi APs | UniFi Network Controller (Docker) |
| pfSense | WAN/ISP uplink, firewall, DNS, DHCP, VPN |

### Broadcast Network (module-broadcast)
| Component | Detail |
|---|---|
| PTP | IEEE 1588, Arista hardware timestamps, ptp4l for SW endpoints |
| AES67 | Professional audio over IP (PTP + IGMP v3 SSM) |
| SMPTE ST 2110 | Video/audio/ancillary over IP |
| VRF RED | Primary media fabric |
| VRF BLUE | Redundant media fabric (SMPTE 2022-7) |
| VRF MGMT | Management with VRF leak to RED/BLUE |
| IGMP | v3 snooping + PIM sparse-mode on Arista |
| QoS | DSCP marking for media/voice traffic classes |

### CI/CD
| Tool | Role |
|---|---|
| GitLab (self-hosted) | Primary Git + CI/CD |
| GitLab Container Registry | Container images (no Harbor needed) |
| GitLab Runner | CI execution (unlimited self-hosted runners) |
| GitHub | Public/open source repos, secondary mirror |
| Trivy | Container + IaC vulnerability scan + SBOM + secret detection |
| Gitleaks | Git secret scanning (pre-commit + pipeline) |
| OWASP Dependency-Check | Application dependency vulnerability scan |
| Commitizen | Standardized conventional commits |
| release-please | Semantic versioning + changelog generation |
| Ed25519 GPG | Signed commits, Vault-backed, annual rotation |

### Identity & Access
| Tool | Role |
|---|---|
| Authentik | SSO provider (OIDC/SAML), self-hosted OSS |
| HashiCorp Vault | Machine secrets (CI/CD, Ansible, services, K8S) |
| Vaultwarden | Human credentials (Bitwarden-compatible, self-hosted) |
| Apache Guacamole | OOB web gateway (RDP/VNC/SSH via browser, behind SSO) |

### Storage & Database
| Tool | Role |
|---|---|
| PostgreSQL (Patroni) | Primary relational DB cluster (GitLab, Vault, NetBox, Authentik, Zabbix) |
| Redis (Sentinel) | Cache + sessions + queues |
| MinIO | S3-compatible object storage (Loki, backups, artifacts) |
| Neo4J Community | Graph CMDB (nodes/edges, semantic queries, AI-queryable) |

### IPAM / CMDB
| Tool | Role |
|---|---|
| NetBox | IPAM + DCIM source of truth (devices, VLANs, IPv4+IPv6, circuits, VRFs) |
| Neo4J | Graph CMDB — dependency/relationship mapping, AI agent queries |

### Monitoring & Observability
| Tool | Role |
|---|---|
| Prometheus | Metrics collection |
| Grafana | Dashboards |
| Loki | Log aggregation (Syslog RFC5424) |
| Zabbix | SNMP/agentd monitoring (network devices, legacy) |
| Icinga2 | Service monitoring (optional, from RTBF reference) |

### Security & Compliance
| Tool | Role |
|---|---|
| Wazuh | SIEM + IDS/IPS + compliance (ISO 27001, NIS1/NIS2 dashboards) |
| OpenSCAP | Automated compliance scanning (CIS, STIG, ISO profiles) |
| Falco | Runtime security (K8S/container behavioral detection) |
| Lynis | Host hardening audit |
| ptp4l Prometheus exporter | PTP monitoring (module-broadcast) |

### Documentation
| Tool | Role |
|---|---|
| Markdown + PlantUML | Source format |
| KROKI | Self-hosted diagram renderer (PlantUML, Mermaid, D3, BPMN, Graphviz) |
| Sphinx | Documentation build (→ PDF, HTML, DOCX) |
| Draw.io | Architecture diagrams |
| ADR (Markdown) | Architecture Decision Records in `adr/` folder |

### Developer Environment
| Tool | Role |
|---|---|
| VSCode | Primary IDE |
| devcontainer | Docker-based dev environment per repo |
| extensions.json | Language-specific extension sets (Go, C++, C#, Python, TypeScript, YAML, Terraform, Ansible, Docker, K8S) |

### VoIP (module-voip)
| Tool | Role |
|---|---|
| Kamailio | SIP proxy/router |
| Janus | WebRTC gateway (SIP ↔ WebRTC bridge) |
| Coturn | STUN/TURN server for WebRTC NAT |
| Homer | SIP/VoIP capture + analysis |

### CCTV (module-cctv)
| Tool | Role |
|---|---|
| Frigate | NVR + AI object detection |
| ZoneMinder | Alternative NVR (more mature) |

---

## Organization Structure

### GitHub (public/open source)
```
by-systems              → root org
by-systems-infra        → infrastructure repos
by-systems-platform     → platform services
by-systems-sec          → security/compliance
by-systems-tpl          → templates
by-systems-doc          → documentation
```

### GitLab self-hosted (private/internal)
```
by-systems/
  infra/
    proxmox-ansible
    network-arista
    network-pfsense
    storage-minio
  platform/
    gitlab-core
    vault-config
    authentik-config
    netbox-config
    monitoring-stack
    neo4j-cmdb
    traefik-config
    guacamole-config
    vaultwarden-config
  sec/
    wazuh-config
    openscap-policies
    compliance-reports
  tpl/
    repo-infra
    repo-app
    repo-svc
    discord-setup
    pipeline-base
  doc/
    adr
    runbooks
    sow-templates
```

Customer projects:
```
{customer-slug}/
  infra/
  platform/
  app/
```

---

## Naming Convention (Draft — formal doc pending)

### Repo pattern
```
{scope}-{component}-{qualifier}
```

Scopes: `infra`, `platform`, `app`, `svc`, `lib`, `tpl`, `doc`, `sec`

### Branch pattern
```
{type}/{issue-id}-{short-description}
feat/42-add-vault-integration
fix/17-dns-resolution-failure
```

### FQDN pattern
```
{service}.{org}.internal          → internal
{service}.{org}.be                → public (if needed)
vault.by-systems.internal
gitlab.by-systems.internal
authentik.by-systems.internal
```

### Environment stages
```
dev → test → staging → acceptance → prod
```
Lite track for simple SME: `dev → staging → prod`

### Document naming
```
{type}-{subject}-{YYYY-MM-DD}.md
adr/0001-platform-stack-decisions.md
sow-infra-network-setup-2026-03-25.md
runbook-vault-backup.md
```

### Commit message (Commitizen)
```
{type}({scope}): {short description}
feat(platform): add vault docker compose stack
fix(infra): correct vlan assignment for iot segment
docs(sec): add adr for gpg key policy
```

Types: feat, fix, chore, docs, test, ci, refactor, perf, security, revert
Scopes: infra, platform, app, svc, lib, tpl, doc, sec

### Device naming (NetBox-aligned)
```
{function}-{type}-{env}-{number}
sw-arista-prod-01        → Arista switch, production, unit 1
ap-unifi-prod-01         → UniFi AP, production, unit 1
srv-proxmox-prod-01      → Proxmox server, production, unit 1
```

### Issue labels
```
type: bug | feature | task | chore | docs | security
scope: infra | platform | app | svc
priority: p0 | p1 | p2 | p3
env: dev | staging | prod
```

---

## GitLab Nginx / Traefik Pattern

GitLab ships with internal Nginx — not removed, but kept internal:
```
gitlab.rb:
  nginx['listen_port'] = 8080
  nginx['listen_https'] = false
  nginx['proxy_set_headers'] = { "X-Forwarded-Proto" => "https" }
```

Traefik handles all external TLS. Single cert management point.
Same pattern applies to all services.

---

## Pending Formal Documents

1. `docs/naming-convention.md` — full naming convention
2. `docs/stack.md` — complete stack inventory table
3. `docs/adr/0001-platform-stack-decisions.md` — all tool choices with rationale

---

## Pending Decisions (minor)

- Sphinx vs MkDocs (Sphinx preferred per user, MkDocs simpler — TBD)
- UniFi Controller vs OpenWRT for WiFi AP management (Ubiquiti hardware confirmed)
- Headscale (self-hosted Tailscale) vs NetBird for mesh VPN (both viable)
- Neo4J D3.js visualization layer vs another graph UI

---

## Files Shared This Session

1. Whiteboard photo — full infra design (Proxmox, K8S, AD, Vault, GitLab, monitoring, ISO 27001, NIS1/NIS2)
2. Draw.io file 1 (`network-infra.drawio`) — 7 tabs: DC-OU, ECAM-DC-OU, FIREWALL-SSO-LDAPS/RADIUS/KERBEROS/VPN, Network infra
3. Draw.io file 2 (`devops-diagrams.drawio`) — 9 tabs: K8S-JAVA, CI_K8S, VMWARE, CI_VMWARE, BARE METAL, C#APPS, INTUNE, NETWORK, OLD-C#APPS
4. PowerPoint (`rtbf_cmdb_applications_de_production_et_de_gestion.pptx`) — 62 slides, RTBF 2024 CMDB audit
5. Corey Ganim PDF — "Anatomy of a perfect OpenClaw setup" guide
