# BY-SYSTEMS — Naming Convention
**Last updated:** 2026-04-01  
**Status:** Draft — living document  
**Scope:** All projects (internal + customer deployments)

> **Key principle:** One slug, three systems.
> A service named `gitlab` in NetBox = FQDN `gitlab.poc.by-systems.be` = repo `platform-gitlab-core`.
> All names use the same format: **lowercase, hyphens, no spaces, no special characters, English only.**

---

## 1. Naming Domains

| Domain | Source of Truth | Format |
|---|---|---|
| Infrastructure (devices, VMs, IPs, VLANs, FQDNs) | **NetBox** | see §6, §7 |
| Services (what runs where) | **NetBox** (custom fields) | see §7 |
| Dependencies + RCA | **Neo4J** | derived from NetBox |
| Repositories | **Naming convention** (this doc) | see §2 |
| Branches | **Naming convention** | see §3 |
| Commits | **Commitizen config** (in repo) | see §4 |
| Documents | **Naming convention** | see §5 |
| Container images | **Naming convention** + CI pipeline | see §8 |
| K8S resources | **Naming convention** + Helm values | see §9 |
| Issue labels | **Naming convention** + GitLab config | see §10 |
| Versioning | **Git tags** + release-please | see §11 |

---

## 2. Repository Naming

### Pattern
```
{scope}-{component}-{qualifier}
```

All lowercase, hyphens only, no underscores.

### Scopes

| Scope | Domain |
|---|---|
| `infra` | Infrastructure (network, compute, storage, hardware) |
| `platform` | Platform services (GitLab, Vault, monitoring, NetBox, etc.) |
| `app` | Application layer (customer-facing apps) |
| `svc` | Microservice |
| `lib` | Shared library |
| `tpl` | Template / boilerplate |
| `doc` | Documentation only repo |
| `sec` | Security / compliance |
| `mod` | Optional platform module (broadcast, voip, cctv) |

### Qualifier (optional)
Used to distinguish variants of the same component:

| Qualifier | Meaning |
|---|---|
| `core` | Main/primary repo |
| `config` | Configuration only |
| `charts` | Helm charts |
| `runner` | CI runner config |
| `api` | Backend API |
| `ui` | Frontend |
| `ansible` | Ansible roles/playbooks |
| `terraform` | Terraform modules |

### Examples

```
platform-gitlab-core
platform-vault-config
platform-monitoring-stack
platform-authentik-config
platform-netbox-config
platform-traefik-config
infra-proxmox-ansible
infra-network-arista
infra-network-pfsense
infra-storage-minio
sec-wazuh-config
sec-openscap-policies
mod-voip-kamailio
mod-voip-rtpengine
mod-broadcast-ptp
mod-cctv-frigate
svc-virtual-sip-codec
svc-kamailio-api
tpl-repo-infra
tpl-repo-app
tpl-pipeline-base
tpl-discord-setup
doc-adr
doc-runbooks
```

### GitLab subgroup structure (self-hosted)
```
org/
  infra/
  platform/
  sec/
  mod/
  svc/
  tpl/
  doc/

{customer-slug}/
  infra/
  platform/
  app/
```

### GitHub org structure (public/open source)
```
org-infra/
org-platform/
org-sec/
org-tpl/
{customer-slug}-infra/
```

### Customer slug format
```
{customer-short-name}
```
Examples: `acme`, `client-a`, `telenet`, `isp1`  
Must match NetBox **tenant slug**.

---

## 3. Branch Naming

### Pattern
```
{type}/{issue-id}-{short-description}
```

Short description: lowercase, hyphens, max 5 words.

### Branch types

| Type | Use |
|---|---|
| `feat` | New feature |
| `fix` | Bug fix |
| `chore` | Maintenance, dependency updates |
| `docs` | Documentation only |
| `hotfix` | Urgent production fix |
| `release` | Release preparation |
| `refactor` | Code restructuring, no behavior change |
| `test` | Test additions or fixes |
| `ci` | CI/CD pipeline changes |
| `sec` | Security fix or hardening |

### Examples
```
feat/42-add-vault-integration
fix/17-dns-resolution-failure
docs/5-adr-gitlab-decision
hotfix/99-kamailio-register-crash
release/1.2.0
chore/31-update-ansible-galaxy
sec/58-rotate-gpg-keys
ci/12-add-trivy-scan-stage
```

### Protected branches
- `main` — production, protected, requires MR + approval
- `develop` — integration branch (optional, per project)
- `release/*` — release candidates, protected

---

## 4. Commit Message Format (Commitizen)

### Pattern
```
{type}({scope}): {short description}

[optional body]

[optional footer: BREAKING CHANGE, Closes #issue]
```

- Type and scope: lowercase
- Short description: imperative mood, no period, max 72 chars
- Body: explain *why*, not *what* (the diff shows what)

### Commit types

| Type | Use |
|---|---|
| `feat` | New feature (bumps MINOR) |
| `fix` | Bug fix (bumps PATCH) |
| `chore` | No production code change |
| `docs` | Documentation only |
| `test` | Test only |
| `ci` | CI/CD pipeline |
| `refactor` | Refactor, no behavior change |
| `perf` | Performance improvement |
| `security` | Security fix or hardening |
| `revert` | Revert previous commit |

### Scopes (aligned to repo scopes)

`infra` | `platform` | `app` | `svc` | `lib` | `tpl` | `doc` | `sec` | `mod`

Sub-scopes allowed: `platform:gitlab`, `infra:network`, `mod:voip`

### Examples
```
feat(platform): add vault docker compose stack
fix(infra): correct vlan assignment for iot segment
docs(sec): add adr for gpg key policy
feat(mod:voip)!: replace siremis with custom kamailio api

BREAKING CHANGE: siremis config format no longer supported
Closes #42
```

### Breaking changes
Add `!` after type/scope OR add `BREAKING CHANGE:` footer.  
Breaking changes bump MAJOR version.

---

## 5. Document and File Naming

### Pattern
```
{type}-{subject}-{YYYY-MM-DD}.md
```

Subject: lowercase, hyphens, descriptive.  
Date: ISO 8601 (`YYYY-MM-DD`).

### Document types

| Type | Description |
|---|---|
| `adr` | Architecture Decision Record |
| `sow` | Scope of Work |
| `rfc` | Request for Comments |
| `runbook` | Operational procedure |
| `changelog` | Auto-generated by release-please |
| `roadmap` | Product roadmap |
| `raid` | Risks, Assumptions, Issues, Dependencies |

### ADR numbering
Sequential per repo, zero-padded to 4 digits:
```
docs/adr/
  0001-use-gitlab-ci.md
  0002-vault-as-secret-manager.md
  0003-traefik-as-edge-proxy.md
  0004-postgresql-patroni-cluster.md
```

### Template files

Template documents use the `.tpl.md` infix — readable as Markdown in VSCode and all editors, with the `.tpl.` infix clearly signaling "not a real document, do not deploy as-is".

```
{NAME}.tpl.md
```

Examples:
```
docs/templates/README.tpl.md
docs/templates/CONTRIBUTING.tpl.md
docs/templates/CLAUDE.tpl.md
docs/templates/sow.tpl.md
```

> Do **not** use `.md.template` (breaks editor Markdown rendering) or `.tmpl` (ambiguous with Go templates).

### Examples
```
docs/adr/0001-platform-stack-decisions.md
docs/sow-infra-network-setup-2026-03-25.md
docs/runbook-vault-backup.md
docs/runbook-gitlab-upgrade.md
docs/raid-platform-poc-2026-q1.md
docs/roadmap-platform-2026-q1.md
docs/templates/README.tpl.md
```

### Diagrams
```
docs/diagrams/
  {subject}.drawio
  {subject}.png          (exported — committed alongside .drawio)
  {subject}.puml         (PlantUML source)
```

---

## 6. FQDN and DNS Naming

### Canonical pattern — split DNS
```
{service}.{env}.by-systems.be
```

> **Decision (2026-04-01):** All service FQDNs use `{service}.{env}.by-systems.be` with split DNS.
> Internal: Pi-hole/OPNsense Unbound resolves to private IP.
> External: Cloudflare resolves to public IP.
> Cert: `*.{env}.by-systems.be` via Let's Encrypt Cloudflare DNS-01.
> NO `.internal`, NO `.arpa` for service FQDNs.

### Examples
```
vault.poc.by-systems.be
gitlab.poc.by-systems.be
authentik.poc.by-systems.be
netbox.poc.by-systems.be
grafana.poc.by-systems.be
prometheus.poc.by-systems.be
guacamole.poc.by-systems.be
kamailio.poc.by-systems.be

gitlab.prod.by-systems.be        (production environment scoped)
gitlab.staging.by-systems.be     (staging)
gitlab.dev.by-systems.be         (dev)

vault.prod.by-systems.be         (public — external access)
vpn.by-systems.be                (public — VPN entry)
```

### DNS ownership
- `*.{env}.by-systems.be` → Pi-hole/OPNsense Unbound (internal resolver)
- `*.{env}.by-systems.be` → Cloudflare DNS-01 (external / cert issuance)
- Split DNS: same FQDN resolves differently inside vs outside

---

## 7. Device and Service Naming (NetBox)

### Device naming
```
{function}-{vendor}-{env}-{number:02d}
```

| Segment | Examples |
|---|---|
| function | `sw` (switch), `ap` (access point), `srv` (server), `fw` (firewall), `rt` (router), `kvm` (KVM) |
| vendor | `arista`, `unifi`, `pfsense`, `proxmox`, `dell`, `hp` |
| env | `prod`, `staging`, `dev`, `mgmt` |
| number | `01`, `02`, ... |

### Examples
```
sw-arista-prod-01
sw-arista-prod-02
ap-unifi-prod-01
srv-proxmox-prod-01
fw-pfsense-prod-01
```

### VM naming
```
vm-{service}-{env}-{number:02d}
```
Examples:
```
vm-gitlab-prod-01
vm-vault-prod-01
vm-kamailio-prod-01
vm-db-prod-01
vm-db-prod-02
```

### Service naming in NetBox
Service name = component slug from repo name.  
Matches FQDN prefix and Prometheus job label.

| NetBox service name | Repo | FQDN | Prometheus job |
|---|---|---|---|
| `gitlab` | `platform-gitlab-core` | `gitlab.poc.by-systems.be` | `gitlab` |
| `vault` | `platform-vault-config` | `vault.poc.by-systems.be` | `vault` |
| `kamailio` | `mod-voip-kamailio` | `kamailio.poc.by-systems.be` | `kamailio` |
| `netbox` | `platform-netbox-config` | `netbox.poc.by-systems.be` | `netbox` |

### NetBox custom fields per service
- `repo_url` — GitLab/GitHub repo link
- `prometheus_job` — Prometheus scrape job name
- `grafana_tag` — Grafana dashboard tag (`service:{name}`)
- `version` — deployed version
- `lifecycle` — `active`, `deprecated`, `eol`

---

## 8. Container Image Tagging

### Pattern
```
{registry}/{scope}/{component}:{version}-{env}
{registry}/{scope}/{component}:{version}        (for release tags)
```

### Registry
```
registry.poc.by-systems.be/{scope}/{component}
```
(GitLab built-in registry, accessible at `gitlab.poc.by-systems.be/registry`)

### Examples
```
registry.poc.by-systems.be/platform/gitlab-core:1.2.3-prod
registry.poc.by-systems.be/mod/voip-kamailio:2.0.1-staging
registry.poc.by-systems.be/svc/virtual-sip-codec:0.3.0
```

### CI tags
- `latest` — latest build on `main` (never use in production manifests)
- `{semver}` — release tag (use in production)
- `{branch}-{short-sha}` — feature branch build

---

## 9. Kubernetes Resource Naming

### Namespaces (per environment)
```
{scope}-{env}
```
Examples:
```
platform-prod
platform-staging
platform-dev
mod-voip-prod
infra-prod
```

### Deployment / Service / ConfigMap naming
```
{component}-{qualifier}
```
Examples:
```
gitlab-core
vault-agent
kamailio-proxy
rtpengine-node-01
postgresql-primary
```

### Labels (mandatory on all resources)
```yaml
labels:
  app.kubernetes.io/name: gitlab
  app.kubernetes.io/component: core
  app.kubernetes.io/version: "1.2.3"
  app.kubernetes.io/managed-by: helm
  by-systems.be/scope: platform
  by-systems.be/env: prod
  by-systems.be/service: gitlab
```

---

## 10. Issue Label Taxonomy

Applied in GitLab and GitHub.

| Dimension | Values |
|---|---|
| `type` | `bug`, `feature`, `task`, `chore`, `docs`, `security`, `question` |
| `scope` | `infra`, `platform`, `app`, `svc`, `sec`, `mod` |
| `priority` | `p0` (critical), `p1` (high), `p2` (medium), `p3` (low) |
| `env` | `dev`, `staging`, `prod` |
| `status` | `blocked`, `in-review`, `needs-info` |

### Label colors (GitLab)
- `type:bug` → red
- `type:feature` → blue
- `priority:p0` → red (bold)
- `priority:p1` → orange
- `scope:sec` → purple
- `env:prod` → dark red

---

## 11. Versioning

### Semantic versioning (SemVer)
```
MAJOR.MINOR.PATCH
```

| Change | Version bump |
|---|---|
| Breaking change (`feat!`, `fix!`, `BREAKING CHANGE`) | MAJOR |
| New feature (`feat`) | MINOR |
| Bug fix, chore, docs (`fix`, `chore`, `docs`) | PATCH |

### Pre-release tags
```
1.0.0-alpha.1
1.0.0-beta.1
1.0.0-rc.1
```

### Git tag format
```
v{semver}
v1.2.3
v2.0.0-rc.1
```

### release-please
Automated via release-please GitHub/GitLab Action:
- reads conventional commits since last tag
- bumps version according to commit types
- generates `CHANGELOG.md` with compare links
- opens release PR → merging = tag + release

---

## 12. GPG & SSH Key Naming

### 12.1 GPG Key UID format
```
{Full Name} ({purpose}) <{email}>
```

Examples:
```
Youssef Boujraf (BY-SYSTEMS DevOps) <y.boujraf@by-systems.be>
GitLab CI Runner (platform-gitlab-prod-01) <ci@by-systems.be>
```

### GPG Key policy
- Algorithm: Ed25519
- Expiry: 1 year (annual rotation)
- Backup: Vault (HashiCorp)
- Hardware token: Yubikey (recommended for senior contributors)

---

### 12.2 SSH Key Naming (Ed25519)

#### Key filename format
```
id_ed25519_{purpose}_{scope}
```

All lowercase, underscores (SSH convention), no hyphens in filename.

#### Examples
```
id_ed25519_personal_by-systems # personal workstation → BY-SYSTEMS infra
id_ed25519_deploy_platform_gitlab    # deploy key for platform-gitlab-core repo
id_ed25519_ci_runner_prod            # GitLab CI runner (prod)
id_ed25519_ansible_infra             # Ansible service account
id_ed25519_backup_borgmatic          # Borgmatic backup agent
```

#### SSH key comment format (inside the key)
```
{purpose}@{scope} {YYYY-MM-DD}
```

Examples:
```
ansible@infra 2026-03-25
ci-runner@platform-gitlab-prod-01 2026-03-25
deploy@platform-gitlab-core 2026-03-25
```

#### SSH Key policy
- Algorithm: Ed25519 only (no RSA, no ECDSA)
- Passphrase: mandatory for human keys; optional for service/deploy keys stored in Vault
- Rotation: annual (align with GPG)
- Storage: human keys on Yubikey (recommended); service keys in HashiCorp Vault (`secret/ssh/{scope}/{purpose}`)
- Deploy keys: one key per repo per environment (no shared deploy keys across repos)
- Authorized keys managed via Ansible (`authorized_key` module) — no manual `~/.ssh/authorized_keys` edits

---

## 13. Environment Names

| Short | Full name | Usage |
|---|---|---|
| `dev` | Development | Local developer, feature branches |
| `test` | Test | Automated tests (unit, integration, security) |
| `staging` | Staging | Full environment, mirrors prod config |
| `acceptance` | Acceptance | UAT, customer sign-off |
| `prod` | Production | Live |

**Lite track** (simple SME): `dev → staging → prod`  
**Full track**: `dev → test → staging → acceptance → prod`

Environment appears in: branch names, FQDNs, K8S namespaces, image tags, Grafana tags, NetBox device names.

---

## 14. Asset Folder Structure & Git LFS

### Scope
Every repo that contains documentation or media uses a standard `assets/` folder. All binary files (images, diagrams, exports, PDFs, videos) are tracked via **Git LFS** — never committed as raw Git objects.

### Folder structure
```
assets/
  diagrams/     # editable source files (.drawio, .puml, .mmd)
  exports/      # rendered outputs (.png, .svg, .pdf) — auto-generated from diagrams/
  images/       # screenshots, photos, logos, UI captures (.png only)
  docs/         # third-party PDFs, vendor datasheets, customer-provided documents
  media/        # screen recordings, demo videos (.mp4, compressed via Handbrake)
```

> `assets/exports/` is what `.md` files link to. Never link directly to `assets/diagrams/`.  
> `assets/media/` files must be compressed before commit (Handbrake, target ≤ 50 MB per file).

### Git LFS tracked extensions (`.gitattributes`)
```
*.png  filter=lfs diff=lfs merge=lfs -text
*.svg  filter=lfs diff=lfs merge=lfs -text
*.pdf  filter=lfs diff=lfs merge=lfs -text
*.drawio filter=lfs diff=lfs merge=lfs -text
*.mp4  filter=lfs diff=lfs merge=lfs -text
*.mov  filter=lfs diff=lfs merge=lfs -text
*.zip  filter=lfs diff=lfs merge=lfs -text
```

### Asset file naming

#### Images & screenshots
```
{subject}-{context}-{YYYY-MM-DD}.png
```
Examples:
```
assets/images/vault-login-ui-2026-03-25.png
assets/images/gitlab-pipeline-failed-2026-03-25.png
assets/images/netbox-rack-dc01-2026-03-25.png
```

#### Diagrams (source)
```
{type}-{subject}-{version}.{ext}
```
Examples:
```
assets/diagrams/arch-platform-overview-v1.drawio
assets/diagrams/seq-vault-auth-flow-v2.puml
assets/diagrams/network-vlan-topology-v1.drawio
```

#### Exports (rendered)
Same base name as source, extension changes:
```
assets/exports/arch-platform-overview-v1.png
assets/exports/seq-vault-auth-flow-v2.png
```

#### Third-party documents
```
{vendor}-{subject}-{YYYY-MM-DD}.pdf
```
Examples:
```
assets/docs/arista-eos-7050x-datasheet-2026-01-15.pdf
assets/docs/customer-client-a-network-diagram-2026-03-01.pdf
assets/docs/hashicorp-vault-reference-arch-2025-11-01.pdf
```

#### Screen recordings
```
{subject}-{context}-{YYYY-MM-DD}.mp4
```
Examples:
```
assets/media/demo-vault-unsealing-2026-03-25.mp4
assets/media/runbook-gitlab-upgrade-walkthrough-2026-03-25.mp4
```

### Screenshot rules
- Format: `.png` only (lossless)
- EXIF stripped: `mogrify -strip assets/images/*.png`
- Standard dimensions: `1920×1080` (full), `1280×720` (panel), `800×600` (component)
- Redaction required if image contains: credentials, tokens, IPs, hostnames, customer data → use Flameshot blur/pixelate before export

### Linking from Markdown
```markdown
![Description](../assets/exports/arch-platform-overview-v1.png)
![Screenshot](../assets/images/vault-login-ui-2026-03-25.png)
[Vendor datasheet](../assets/docs/arista-eos-7050x-datasheet-2026-01-15.pdf)
```

Relative paths only. No absolute paths, no external image URLs in committed docs.

---

## 15. Mail Infrastructure

### Strategy

| Environment track | Mail server | Domain | Notes |
|---|---|---|---|
| `dev` / `test` / `staging` / `acceptance` | **Mailcow** (self-hosted) | `example.com` (internal) | Full stack: SMTP, IMAP, CalDAV, CardDAV |
| `prod` | External provider | real domain | Microsoft Exchange, Google Workspace, or equivalent |

> **Why `example.com` for non-prod?**  
> RFC 2606 reserves `example.com` — safe to use internally with no risk of leaking mail externally. Mailcow is configured to be authoritative for this domain on the internal DNS (`*.poc.by-systems.be`).

---

### Mailcow (non-prod)

**Service name:** `platform-mailcow-{env}` (e.g. `platform-mailcow-dev`)  
**FQDN:** `mail.example.com` (internal DNS only)  
**Webmail:** `https://mail.example.com`  
**SMTP:** `smtp.example.com:587` (STARTTLS)  
**IMAP:** `imap.example.com:993` (TLS)  
**CalDAV/CardDAV:** `https://mail.example.com` (via SOGo)

#### Test domains per environment

| Environment | Domain | Notes |
|---|---|---|
| `dev` | `dev.example.com` | developer local testing |
| `test` | `test.example.com` | automated pipeline testing |
| `staging` | `staging.example.com` | full env, mirrors prod |
| `acceptance` | `acceptance.example.com` | UAT / customer sign-off |

#### Test mailboxes naming
```
{role}@{env}.example.com
```

Examples:
```
admin@dev.example.com
noreply@staging.example.com
ci-notify@test.example.com
user-01@acceptance.example.com
```

#### Client configuration (Thunderbird)
- Thunderbird profiles per environment (dev/test/staging/acceptance)
- Auto-discover via `autoconfig.example.com` (Mailcow built-in)
- Used for: testing notification pipelines, GitLab email alerts, Authentik account invites, Grafana alerts, etc.

---

### Production mail

- Use customer's existing provider (Exchange, Google Workspace, etc.)
- Outbound relay from platform services (GitLab, Grafana, Authentik) via SMTP relay → customer's SMTP
- No self-hosted mail in production unless explicitly scoped

---

### Repo name
`platform-mailcow-core` — Mailcow Docker Compose + Ansible role + Traefik integration

---

## 16. Update Process

1. Propose change via MR to `doc-adr` or the repo containing this file
2. If change affects NetBox slugs → update NetBox tenant/service names accordingly
3. If change affects K8S labels → update Helm chart defaults
4. If change affects CI → update `.commitlintrc` / `cz.toml` in `tpl-pipeline-base`
5. Communicate to team before merging

This document is the **single reference**. When in doubt, check here first.
