# Platform Roadmap

> **Last updated:** 2026-04-14
> **Maintained by:** Rune — update when a layer advances or a scoped ADR changes the plan
> **Reading order:** [`status.md`](status.md) → this file → [`docs/adr/README.md`](adr/README.md)

Forward-looking delivery plan for the BY-SYSTEMS platform. Structure follows the 6-layer model in [`infra/0002-platform-charter §Layer model`](adr/infra/0002-platform-charter.md). Each layer must be documented, tested, and signed off before the next begins, with the parallel-development exception documented in the same ADR.

For live state (what is deployed today vs pending), see [`status.md`](status.md). This file is the *plan*, not the state.

---

## Layer model at a glance

| Layer | Name | Core ADRs |
|---|---|---|
| 0 | Standards & Templates | [`infra/0002-platform-charter`](adr/infra/0002-platform-charter.md), naming scope, [`OPERATING-STANDARD.md`](../OPERATING-STANDARD.md) |
| 1 | Proxmox Base | [`infra/0001-platform-stack`](adr/infra/0001-platform-stack.md), [`infra/0003-terraform-standard`](adr/infra/0003-terraform-standard.md), [`identity/0004-os-accounts`](adr/identity/0004-os-accounts.md), [`security/0003-hardening`](adr/security/0003-hardening.md) |
| 2 | Vault + step-ca | [`security/0001-secret-storage`](adr/security/0001-secret-storage.md), [`security/0004-certificate-strategy`](adr/security/0004-certificate-strategy.md) |
| 3 | Identity (Authentik) | [`identity/0001-authentication`](adr/identity/0001-authentication.md), [`identity/0002-provisioning`](adr/identity/0002-provisioning.md), [`identity/0003-machine-credentials`](adr/identity/0003-machine-credentials.md) |
| 4 | Storage (Synology + MinIO) | [`infra/0004-network-architecture`](adr/infra/0004-network-architecture.md), [`infra/0008-backup-strategy`](adr/infra/0008-backup-strategy.md) |
| 5 | Platform Services | [`services/0001-opnsense`](adr/services/0001-opnsense.md), [`services/0002-email-infrastructure`](adr/services/0002-email-infrastructure.md), [`services/0003-netbox-cmdb`](adr/services/0003-netbox-cmdb.md), [`services/0004-database-strategy`](adr/services/0004-database-strategy.md), [`services/0005-notifications`](adr/services/0005-notifications.md) |

---

## Layer 0 — Standards & Templates ✅

**Status:** Complete (2026-04-14). Covered by the 30 scoped ADRs and `OPERATING-STANDARD.md`. Remaining work is maintenance: keep templates in `docs/templates/` aligned with backport improvements from live repos.

---

## Layer 1 — Proxmox Base ⚠ Partial

**Goal:** A reproducible Proxmox host baseline with hardened OS accounts, automated provisioning via Terraform + Ansible, and a cloud-init golden image.

- [x] `srv-proxmox-poc-01` physical node operational
- [x] `ansible-platform/roles/hardening` — sshd template, fail2ban, ufw, postfix live per [`identity/0004-os-accounts §5`](adr/identity/0004-os-accounts.md)
- [x] Terraform state backup to Synology NAS after every apply/destroy per [`infra/0003-terraform-standard §State backend evolution`](adr/infra/0003-terraform-standard.md)
- [ ] `ansible-platform/roles/user-mgmt` — ⚠ pending verify
- [ ] `ansible-platform/roles/git-config` — pending write per [`git/0003-configuration`](adr/git/0003-configuration.md)
- [ ] `ansible-platform/roles/key-mgmt` — pending write (SSH/GPG key distribution)
- [ ] Debian cloud-init golden template — ⚠ pending verify
- [ ] 6 pending thresholds in [`security/0003-hardening §Pending decisions`](adr/security/0003-hardening.md) resolved (patch cadence, Lynis score, Trivy CVE gate, filesystem policy, retention, Lynis scope)
- [ ] Production Proxmox node signed off against hardening baseline

**Exit criteria:** VMs provision via Terraform, hardened by Ansible on first boot, OS accounts per `identity/0004`, state backed up to NAS. All 6 `security/0003` thresholds locked.

---

## Layer 2 — Vault + step-ca ❌ Not started

**Goal:** Single source of machine secrets and internal TLS. Unblocks every downstream layer.

- [ ] Deploy HashiCorp Vault per [`security/0001-secret-storage`](adr/security/0001-secret-storage.md)
  - Seal/unseal strategy, KV v2 paths per `§KV path convention`
  - AppRole + Kubernetes auth methods (when K8s arrives)
  - Transitional JSON secret state in `infra/secrets/` retired per `§Transitional state`
- [ ] Deploy step-ca per [`security/0004-certificate-strategy`](adr/security/0004-certificate-strategy.md)
  - Root CA + intermediate CA established
  - ACME provisioner for internal hostnames
- [ ] `ansible-platform/roles/ca-trust` — writes root CA to every VM's trust store per `security/0004 §Root CA distribution` (silent HTTPS breakage if skipped)
- [ ] Vault + step-ca backed up per [`infra/0008-backup-strategy`](adr/infra/0008-backup-strategy.md) (once the 11 pending thresholds are resolved)
- [ ] `svc-rune` API key migrated from `infra/secrets/` to Vault

**Exit criteria:** All machine secrets live in Vault. Every VM trusts the internal root CA. No service uses a plaintext secret on disk.

---

## Layer 3 — Identity (Authentik) ❌ Not started

**Goal:** Authentik is the IAM hub. Every human and service identity flows through it.

- [ ] Deploy Authentik per [`identity/0001-authentication`](adr/identity/0001-authentication.md)
- [ ] Provisioning flows per [`identity/0002-provisioning`](adr/identity/0002-provisioning.md) — Authentik → downstream tools
- [ ] Machine credential pattern `{IDENTITY}_{PLATFORM}_TOKEN` enforced per [`identity/0003-machine-credentials`](adr/identity/0003-machine-credentials.md)
- [ ] MFA (TOTP) required on all human accounts
- [ ] OIDC clients registered for Layer 5 services as they come online

**Exit criteria:** All platform services authenticate via Authentik OIDC/SAML. No local accounts on Layer 5 services.

---

## Layer 4 — Storage ⚠ Partial

**Goal:** Shared storage for VM disks, object storage for artifacts/backups, backup target for Layer 2 secrets.

- [x] Synology DS1513+ operational — Terraform state backup target
- [x] `lib-synology-dsm` ships FileStation, SystemManager, SharePermissionManager (exact version + `lib/python/0001-design-standard` compliance ⚠ pending audit)
- [ ] MinIO deployed for S3-compatible object storage
- [ ] Backup policy fully automated — 11 pending thresholds in [`infra/0008-backup-strategy §Pending decisions`](adr/infra/0008-backup-strategy.md) (off-site target, schedules, retention, RTO, drill cadence, KMS)
- [ ] Off-site backup target chosen and operational
- [ ] Restore drill executed and documented

**Exit criteria:** Shared + object storage operational. Backup policy fully implemented against locked thresholds. First restore drill passed.

---

## Layer 5 — Platform Services ❌ Not started (except OPNsense test instance)

**Goal:** The services that make the platform useful: firewall/DHCP/DNS (OPNsense), CMDB (NetBox), source forge (GitLab CE), human password manager (Vaultwarden), email (Mailcow), observability, reverse proxy (Traefik), HA databases.

OPNsense is documented as an **exception, not a pattern** per [`services/0001-opnsense`](adr/services/0001-opnsense.md) — six constraints break the generic Linux VM pipeline.

### 5.1 OPNsense (firewall / DHCP / DNS / VPN)

- [x] Test instance `vm-opnsense-01` on test VLANs (OPNsense 26.1.5)
- [x] `lib-opnsense v1.0.0` — 54 managers, 1178 unit + 294 integration tests
- [x] `ansible-opnsense` — 54 modules, one per manager
- [ ] **Production OPNsense deployment** — top blocker (see [`status.md §Top blockers`](status.md))
- [ ] `platform-setup/opnsense/bootstrap.md` runbook — pending write
- [ ] Production VLAN registry applied per [`infra/0004-network-architecture`](adr/infra/0004-network-architecture.md)

### 5.2 NetBox (CMDB — source of truth)

- [ ] Deploy NetBox per [`services/0003-netbox-cmdb`](adr/services/0003-netbox-cmdb.md)
- [ ] 19-role vocabulary loaded
- [ ] Mandatory custom fields enforced (`env`, `repo_url`, `prometheus_job`, etc.)
- [ ] Inventory plugin `netbox.netbox` wired into Ansible
- [ ] Hostnames and prefixes seeded per [`naming/0001-infra`](adr/naming/0001-infra.md)

### 5.3 Databases (HA from day 1)

Per [`services/0004-database-strategy`](adr/services/0004-database-strategy.md), PostgreSQL and Redis run as HA clusters from the start — not single instances later upgraded.

- [ ] PostgreSQL Patroni 3-node cluster
- [ ] Redis with Sentinel
- [ ] 4 deferred decisions resolved: pooler choice (pgbouncer vs pgpool-II), Patroni DCS (etcd vs Consul), Sentinel colocation, Redis replica count
- [ ] Software HA limits documented per `services/0004 §Cluster placement` (protects against process/OS-level failure, not host or site)

### 5.4 Reverse Proxy & TLS edge

- [ ] Deploy Traefik v3
  - Internal TLS from step-ca (Layer 2)
  - Public TLS via Let's Encrypt DNS-01 per [`security/0004 §Public vs internal`](adr/security/0004-certificate-strategy.md)
- [ ] HTTPS-only, no plaintext HTTP exposed

### 5.5 GitLab CE + Nexus Repository OSS

- [ ] Deploy GitLab CE (external PostgreSQL from 5.3, external Redis from 5.3, MinIO from Layer 4)
  - No `:latest` tags — pin to specific versions
  - Container Registry + Package Registry enabled
- [ ] GitLab Runners registered (Docker executor)
- [ ] Deploy Nexus Repository OSS as universal proxy/cache (npm, PyPI, Maven, Go, Docker Hub, Helm)
- [ ] Migrate from GitHub per [`git/0002-platform-strategy`](adr/git/0002-platform-strategy.md)

### 5.6 Vaultwarden (human credentials)

- [ ] Deploy Vaultwarden per [`security/0001-secret-storage §Vaultwarden`](adr/security/0001-secret-storage.md) — SSO via Authentik
- [ ] Machine secrets remain in Vault; humans use Vaultwarden

### 5.7 Email infrastructure

- [ ] Deploy Mailcow per [`services/0002-email-infrastructure`](adr/services/0002-email-infrastructure.md) — standalone/hybrid mode, official image, **internal** MariaDB + Redis (not shared — documented exception to 5.3)
- [ ] `platform-setup/mailcow/lifecycle.md` runbook — pending write
- [ ] SPF / DKIM / DMARC records published

### 5.8 Observability

- [ ] Deploy Loki + Promtail per [`infra/0006-logging`](adr/infra/0006-logging.md) — no PII, structlog JSON from all services
- [ ] Deploy Prometheus + Grafana + Alertmanager per [`infra/0007-monitoring`](adr/infra/0007-monitoring.md)
- [ ] `ansible-platform/roles/observability-client` — installs Promtail + node_exporter on every VM
- [ ] Event routing per [`services/0005-notifications`](adr/services/0005-notifications.md) — `🔔 [Tool] Event` template, Discord one-way

### 5.9 Compliance & GRC

- [ ] Deploy `ciso-assistant-community` (intuitem) as the compliance control registry per [`security/0002-compliance-mapping`](adr/security/0002-compliance-mapping.md)
- [ ] Target frameworks loaded: ISO 27001, NIS2, DORA, GDPR
- [ ] Every ADR's `CISO mapping` section reconciled against loaded controls

---

## Cross-cutting tracks

These run in parallel with layer work per the **parallel development exception** in [`infra/0002-platform-charter §Parallel development exception`](adr/infra/0002-platform-charter.md).

- **Library development** — `lib-opnsense` (v1.0.0 ✅), `lib-synology-dsm` (pending audit), future per-tool libraries per [`lib/python/0001-design-standard`](adr/lib/python/0001-design-standard.md)
- **Ansible collections** — `ansible-opnsense` (54 modules ✅), future collections as libraries land
- **Terraform modules** — `infra-terraform-proxmox` module family per [`infra/0003-terraform-standard`](adr/infra/0003-terraform-standard.md)
- **Runbooks** — `platform-setup` accumulates one runbook per tool as services go live
- **Cross-repo `CLAUDE.md` updates** — backport pattern from the doc-platform-core template changes

---

## How to update

When a layer advances or a scoped ADR changes the plan:

1. Move items between layers' checkboxes as they complete
2. Update [`status.md`](status.md) in the same change — roadmap is the *plan*, status is the *state*
3. Never reference flat ADR numbers — always use scoped paths (`{scope}/NNNN-title`)
4. Commit: `docs: update roadmap to YYYY-MM-DD`
