# ORG Platform — Architecture

**Last updated:** 2026-03-25
**Status:** Draft — PoC phase
**Related docs:** `docs/stack.md`, `docs/naming-convention.md`, `docs/adr/0001-platform-stack-decisions.md`

> Diagrams are written in PlantUML. Rendered via KROKI (`kroki.org.internal`).
> Source files: `assets/diagrams/` — exports: `assets/exports/`

---

## 1. Platform Overview

High-level view of all platform layers and their relationships.

```plantuml
@startuml platform-overview
!theme plain
skinparam backgroundColor #FAFAFA
skinparam defaultFontName monospace
skinparam linetype ortho

title ORG Platform — Overview

together {
  package "Edge & Network" {
    [pfSense CE] as pf
    [WireGuard / NetBird] as vpn
    [Traefik v3] as traefik
    [Bind 9 (internal DNS)] as dns
    [step-ca (internal CA)] as ca
  }
}

together {
  package "Identity & Access" {
    [Authentik (SSO)] as authentik
    [HashiCorp Vault OSS] as vault
    [Vaultwarden] as vaultwarden
    [Teleport CE (bastion)] as teleport
    [Apache Guacamole] as guacamole
  }
}

together {
  package "Compute" {
    [Proxmox VE] as proxmox
    [k3s (Kubernetes)] as k3s
  }
}

together {
  package "CI/CD & Registries" {
    [GitLab CE] as gitlab
    [GitLab Runner] as runner
    [Kaniko (image builder)] as kaniko
    [GitLab Container Registry] as gcr
    [GitLab Package Registry] as gpr
    [Nexus OSS (proxy/cache)] as nexus
  }
}

together {
  package "Data & CMDB" {
    [PostgreSQL (Patroni)] as pg
    [Redis (Sentinel)] as redis
    [MinIO (S3)] as minio
    [NetBox (IPAM/CMDB)] as netbox
    [Neo4J (dependency graph)] as neo4j
  }
}

together {
  package "Observability" {
    [Prometheus] as prom
    [Grafana] as grafana
    [Loki] as loki
    [Zabbix] as zabbix
    [OpenTelemetry Collector] as otel
  }
}

together {
  package "Security" {
    [Wazuh (SIEM/XDR)] as wazuh
    [Falco (runtime)] as falco
    [Trivy (scan)] as trivy
    [Gitleaks] as gitleaks
    [Presidio (PII redact)] as presidio
  }
}

together {
  package "Shared Storage & Collab" {
    [Nextcloud] as nextcloud
    [Mailcow (non-prod)] as mailcow
  }
}

' Edge flows
pf --> traefik : route
traefik --> authentik : SSO
traefik --> gitlab
traefik --> vault
traefik --> netbox
traefik --> grafana
traefik --> teleport

' Auth flows
authentik --> gitlab : OIDC
authentik --> vault : OIDC
authentik --> teleport : OIDC
authentik --> guacamole : OIDC
authentik --> nextcloud : OIDC

' Compute
proxmox --> k3s : VMs/LXC

' CI/CD
gitlab --> runner : triggers
runner --> kaniko : build
kaniko --> gcr : push image
runner --> gpr : publish package
runner --> nexus : resolve deps
nexus ..> gpr : internal pkg fallback

' Data
gitlab --> pg
vault --> pg
netbox --> pg
authentik --> pg
gitlab --> redis
gitlab --> minio : artifacts/LFS

' Observability
prom --> grafana
loki --> grafana
otel --> prom
otel --> loki
wazuh --> grafana

' Security
trivy --> gitlab : scan results
gitleaks --> gitlab : scan results
falco --> wazuh : alerts

@enduml
```

---

## 2. Network Topology

Logical network segmentation — VLANs, zones, and traffic flows.

```plantuml
@startuml network-topology
!theme plain
skinparam backgroundColor #FAFAFA
skinparam defaultFontName monospace
skinparam linetype ortho

title ORG Network — Logical Topology

cloud "Internet / ISP" as internet

package "DMZ (VLAN 10)" {
  [Traefik v3\nedge.org.example] as traefik_dmz
  [Cloudflare DNS\norg.example] as cf
}

package "Management (VLAN 20)" {
  [pfSense CE\nfw-pfsense-prod-01] as pfsense
  [Bind 9\ndns.org.internal] as dns
  [step-ca\nca.org.internal] as ca
  [Proxmox VE\nsrv-proxmox-prod-01] as proxmox
}

package "Platform (VLAN 30)" {
  [GitLab CE\ngitlab.org.internal] as gitlab
  [Vault\nvault.org.internal] as vault
  [Authentik\nauth.org.internal] as authentik
  [NetBox\nnetbox.org.internal] as netbox
  [Nexus OSS\nnexus.org.internal] as nexus
  [Teleport CE\nbastion.org.internal] as teleport
}

package "Observability (VLAN 40)" {
  [Prometheus\nprometheus.org.internal] as prom
  [Grafana\ngrafana.org.internal] as grafana
  [Loki\nloki.org.internal] as loki
  [Wazuh\nwazuh.org.internal] as wazuh
}

package "Workloads / K8S (VLAN 50)" {
  [k3s cluster\n*.svc.cluster.local] as k3s
}

package "Storage (VLAN 60)" {
  [PostgreSQL\npg.org.internal] as pg
  [MinIO\nminio.org.internal] as minio
  [Nextcloud\ncloud.org.internal] as nextcloud
}

package "User devices (VLAN 99)" {
  [Workstations\n(VPN: NetBird/WireGuard)] as ws
}

' Internet flows
internet --> cf : DNS
internet --> pfsense : WAN
pfsense --> traefik_dmz : HTTPS 443

' Internal routing
traefik_dmz --> platform : route *.org.internal
pfsense --> management : admin
ws --> pfsense : VPN tunnel
ws --> teleport : SSH/K8S/DB (bastion)

' Platform to data
gitlab --> pg
vault --> pg
authentik --> pg
netbox --> pg
gitlab --> minio

' DNS
dns --> pfsense : upstream
pfsense ..> dns : split DNS

' Firewall rules (dashed = restricted)
pfsense ..> platform : filtered
pfsense ..> observability : filtered
pfsense ..> workloads : filtered

note bottom of ws
  No direct SSH to VMs.
  All access via Teleport bastion
  or Guacamole (Windows RDP/VNC).
end note

note right of nexus
  GOPROXY, npm, NuGet,
  PyPI, Conan, Maven.
  Cache-first — no direct
  internet from CI builds.
end note

@enduml
```

---

## 3. CI/CD Pipeline Flow

End-to-end flow from git push to deployed service.

```plantuml
@startuml cicd-pipeline
!theme plain
skinparam backgroundColor #FAFAFA
skinparam defaultFontName monospace

title ORG CI/CD Pipeline — End-to-End Flow

actor Developer as dev
participant "Workstation\n(VSCode + Commitizen)" as ws
participant "GitLab CE\ngitlab.org.internal" as gitlab
participant "GitLab Runner\n(Docker executor)" as runner
participant "Nexus OSS\n(dep proxy/cache)" as nexus
participant "Kaniko\n(image builder)" as kaniko
participant "GitLab CR\n(image storage)" as gcr
participant "GitLab PR\n(package storage)" as gpr
participant "Trivy / Gitleaks\n(scanners)" as scan
participant "k3s\n(Kubernetes)" as k3s
participant "Vault\n(secrets)" as vault

== Commit ==
dev -> ws : write code
ws -> ws : pre-commit hooks\n(Gitleaks, lint, commitlint)
ws -> gitlab : git push (signed, Ed25519)

== Pipeline triggered ==
gitlab -> runner : trigger pipeline\n(tpl-pipeline-base)

== Stage 1: Lint & Validate ==
runner -> runner : commitlint, shellcheck\nyaml lint, terraform validate

== Stage 2: Dependency resolution ==
runner -> nexus : resolve deps\n(npm/NuGet/Go/PyPI/Conan)
nexus --> runner : cached or fetched\nfrom upstream

== Stage 3: Build ==
runner -> kaniko : build container image\n(rootless, no Docker daemon)
kaniko -> nexus : RUN steps fetch deps via Nexus
kaniko -> gcr : push image\nregistry.org.internal/{project}:{sha}

== Stage 4: Scan ==
runner -> scan : trivy image (CVE + secrets)\ngitleaks (git history)\nowasp dep-check
scan --> gitlab : results + SBOM\n(block on CRITICAL)

== Stage 5: Test ==
runner -> runner : unit tests\nintegration tests\n(test env, example.com mail)

== Stage 6: Package publish (if lib) ==
runner -> gpr : publish package\n(npm / NuGet / PyPI / Helm)\nvia CI_JOB_TOKEN

== Stage 7: Deploy ==
runner -> vault : fetch deploy secrets\n(AppRole, short-lived)
vault --> runner : secrets (env vars, certs)
runner -> k3s : helm upgrade --install\n(image from GitLab CR)
k3s --> gitlab : deployment status

== Post-deploy ==
k3s -> vault : runtime secrets\n(Vault Agent sidecar)
k3s --> runner : health check OK

gitlab -> dev : pipeline result\n(pass / fail + report)

note over nexus
  First fetch cached permanently.
  Upstream removal has no impact.
  GOPROXY, npm, NuGet, PyPI,
  Conan, Maven all covered.
end note

note over kaniko
  No --build-arg secrets.
  Registry auth via config.json.
  Build secrets via tmpfs mount.
  Trivy validates post-build.
end note

@enduml
```

---

## 4. Access & Identity Flow

How authentication and access control flows from user to service.

```plantuml
@startuml access-flow
!theme plain
skinparam backgroundColor #FAFAFA
skinparam defaultFontName monospace
skinparam linetype ortho

title ORG — Access & Identity Flow

actor "Engineer\n(DevOps)" as eng
actor "Non-technical\nuser" as ntu
actor "CI/CD\nRunner" as ci

package "VPN Layer" {
  [WireGuard / NetBird\n(OIDC: Authentik)] as vpn
}

package "Edge" {
  [pfSense CE\n(firewall + routing)] as pf
  [Traefik v3\n(reverse proxy + TLS)] as traefik
}

package "Identity" {
  [Authentik\n(SSO: OIDC/SAML)] as authentik
  [HashiCorp Vault\n(machine secrets)] as vault
  [Vaultwarden\n(human credentials)] as vw
}

package "Bastion" {
  [Teleport CE\n(cert SSH + K8S + DB)] as teleport
  [Apache Guacamole\n(browser RDP/VNC)] as guacamole
}

package "Services" {
  [GitLab] as gitlab
  [Grafana] as grafana
  [NetBox] as netbox
  [K8S (k3s)] as k3s
  [PostgreSQL] as pg
  [Linux VMs] as vms
  [Windows VMs] as wvms
}

' Engineer flow
eng --> vpn : connect (OIDC)
vpn --> pf : tunnel
eng --> traefik : HTTPS
traefik --> authentik : SSO redirect
authentik --> eng : token
eng --> teleport : SSH / kubectl / psql
teleport --> vms : cert SSH (no shared keys)
teleport --> k3s : kubectl proxy
teleport --> pg : DB proxy

' Non-technical user flow
ntu --> vpn : connect
ntu --> guacamole : browser RDP/VNC
guacamole --> wvms : RDP
guacamole --> vms : VNC/SSH (supervised)

' CI/CD flow
ci --> vault : AppRole auth
vault --> ci : short-lived secrets
ci --> gitlab : CI_JOB_TOKEN
ci -[dashed]-> vms : no direct SSH\n(Teleport only)

' SSO wiring
authentik --> gitlab : OIDC
authentik --> vault : OIDC
authentik --> teleport : OIDC
authentik --> guacamole : OIDC
authentik --> grafana : OIDC
authentik --> netbox : OIDC
authentik --> vpn : OIDC

note bottom of teleport
  All sessions recorded + replayable.
  ISO 27001 / NIS2 audit trail.
  No direct port 22 access
  (pfSense blocks it).
end note

note bottom of vault
  Machine secrets only.
  Human credentials → Vaultwarden.
  CI secrets: AppRole, short-lived.
  K8S secrets: Vault Agent sidecar.
end note

@enduml
```

---

## 5. Module Architecture (Tier 2)

Optional modules layered on top of the base platform.

```plantuml
@startuml module-architecture
!theme plain
skinparam backgroundColor #FAFAFA
skinparam defaultFontName monospace
skinparam linetype ortho

title ORG — Tier 1 Base + Tier 2 Optional Modules

package "Tier 1 — Base Platform (all deployments)" {
  [Compute: Proxmox + k3s] as compute
  [Network: pfSense + Traefik + Bind9] as network
  [Identity: Authentik + Vault + Teleport] as identity
  [CI/CD: GitLab + Kaniko + Nexus] as cicd
  [Observability: Prometheus + Grafana + Loki] as obs
  [Security: Wazuh + Trivy + Falco] as sec
  [Storage: PostgreSQL + MinIO + Nextcloud] as storage
  [CMDB: NetBox + Neo4J] as cmdb
}

package "Tier 2 — Module: Broadcast" #LightBlue {
  [PTP (IEEE 1588v2)] as ptp
  [AES67 (audio over IP)] as aes67
  [SMPTE ST 2110 (video/audio/meta)] as st2110
  [VRF RED (primary media network)] as vrfred
  [VRF BLUE (redundant media network)] as vrfblue
  [Arista EOS (multicast routing)] as arista
}

package "Tier 2 — Module: VoIP" #LightGreen {
  [Kamailio (SIP proxy)] as kamailio
  [RTPEngine (media relay)] as rtpengine
  [Coturn (STUN/TURN)] as coturn
  [Homer (SIP capture)] as homer
  [FreeSWITCH / Asterisk (optional PBX)] as pbx
}

package "Tier 2 — Module: CCTV" #LightYellow {
  [Frigate (NVR + AI detection)] as frigate
  [IP Cameras (ONVIF/RTSP)] as cameras
  [ZoneMinder (fallback NVR)] as zm
}

' Base dependencies
compute --> network
network --> identity
identity --> cicd
cicd --> storage
obs --> storage
cmdb --> storage

' Module dependencies on Tier 1
arista --> network : VLAN/VRF config
ptp --> arista : sync
aes67 --> ptp
st2110 --> ptp
vrfred --> arista
vrfblue --> arista

kamailio --> identity : SSO/auth
kamailio --> obs : metrics
rtpengine --> kamailio
homer --> kamailio : SIP capture

frigate --> storage : recordings → MinIO
frigate --> obs : metrics/alerts
cameras --> frigate : RTSP

note right of "Tier 2 — Module: Broadcast"
  Broadcast module requires
  Arista switches + dedicated
  media network (RED/BLUE VRFs).
  PTP grandmaster required.
end note

note right of "Tier 2 — Module: VoIP"
  VoIP module can run on
  standard network (no Arista).
  Coturn required for WebRTC NAT traversal.
end note

@enduml
```

---

## Diagram source files

| Diagram | Source | Export |
|---|---|---|
| Platform overview | `assets/diagrams/arch-platform-overview-v1.puml` | `assets/exports/arch-platform-overview-v1.png` |
| Network topology | `assets/diagrams/arch-network-topology-v1.puml` | `assets/exports/arch-network-topology-v1.png` |
| CI/CD pipeline | `assets/diagrams/arch-cicd-pipeline-v1.puml` | `assets/exports/arch-cicd-pipeline-v1.png` |
| Access & identity | `assets/diagrams/arch-access-identity-v1.puml` | `assets/exports/arch-access-identity-v1.png` |
| Module architecture | `assets/diagrams/arch-module-tiers-v1.puml` | `assets/exports/arch-module-tiers-v1.png` |

> To render locally: install KROKI CLI or use `kroki.org.internal`.
> To export PNG: `kroki convert assets/diagrams/arch-*.puml --format png --output assets/exports/`
