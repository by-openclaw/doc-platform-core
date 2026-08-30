# Platform RAID Log

> **Last updated:** 2026-08-29
> **Maintained by:** Rune — update on every sprint/phase review or when a blocker changes state
> **Scope:** Platform-wide RAID per [`OPERATING-STANDARD.md §7.1`](../OPERATING-STANDARD.md). Per-repo RAIDs live in each repo's own `RAID.md`.

RAID = **R**isks · **A**ssumptions · **I**ssues · **D**ependencies.

Items that are now formally captured in a scoped ADR's `§Pending decisions` or `§Deferred decisions` section are tracked there, not duplicated here. This file holds cross-cutting items that don't belong to a single ADR.

---

## R — Risks (Open)

| ID | Risk | Severity | Mitigation | Source / ADR |
|---|---|---|---|---|
| R-01 | Single Proxmox host — software HA only; host/site failure unprotected | 🔴 Critical | Documented in [`services/0004-database-strategy §Cluster placement`](adr/services/0004-database-strategy.md). Second host required for hardware HA. | [`services/0004`](adr/services/0004-database-strategy.md) |
| R-03 | HashiCorp Vault seal/unseal — data loss if node lost without backup | 🔴 Critical | Seal strategy + daily snapshot to NAS, tracked in [`infra/0008-backup-strategy §Pending decisions`](adr/infra/0008-backup-strategy.md) (11 TBDs). Blocks Layer 2 sign-off. | [`security/0001`](adr/security/0001-secret-storage.md), [`infra/0008`](adr/infra/0008-backup-strategy.md) |
| R-05 | Kaniko `--build-arg` secret leak in CI | 🔴 Critical | Policy forbids `--build-arg` for secrets. Gitleaks + Trivy post-build catch leaks. | [`security/0003-hardening`](adr/security/0003-hardening.md) |
| R-10 | Authentik SSO outage blocks all platform access | 🔴 Critical | Break-glass local accounts per [`identity/0004-os-accounts §5`](adr/identity/0004-os-accounts.md). Authentik HA required before Layer 3 sign-off. | [`identity/0001`](adr/identity/0001-authentication.md), [`identity/0004`](adr/identity/0004-os-accounts.md) |
| R-14 | PII leak via unredacted screenshots/logs in Git | 🔴 Critical | Presidio pre-commit + CI scan per [`security/0003-hardening`](adr/security/0003-hardening.md). Redaction format standard enforced. | [`security/0003`](adr/security/0003-hardening.md) |
| R-18 | 3com/HPE WAN switch: Telnet enabled — cleartext credentials | 🔴 Critical | Disable Telnet, SSH only. Tracked until OPNsense production deployment replaces the temporary network path. | [`infra/0004-network-architecture`](adr/infra/0004-network-architecture.md) |
| R-19 | Synology NFS shared between prod and non-prod Proxmox | 🔴 Critical | Dedicated non-prod NFS exports required before any non-prod VM mounts NAS. | [`infra/0004`](adr/infra/0004-network-architecture.md) |
| R-06 | Nexus OSS disk exhaustion from cache growth | 🟡 Medium | Blob store cleanup policy (30-day unused removal). Disk >80% alert via Prometheus. | [`infra/0001-platform-stack`](adr/infra/0001-platform-stack.md) |
| R-07 | GitLab LFS storage growth | 🟢 Low | LFS → MinIO backend. Per-project quota. | [`git/0002-platform-strategy`](adr/git/0002-platform-strategy.md) |
| R-08 | Anthropic/OpenAI API overload during heavy agent use | 🟡 Medium | Fallback chain (Anthropic → OpenAI). Monitor via OpenClaw `/stats`. | — |
| R-12 | Nexus upstream proxy blocked by corporate firewall / ISP | 🟡 Medium | OPNsense egress rule explicitly allows Nexus outbound. | [`services/0001-opnsense`](adr/services/0001-opnsense.md) |
| R-20 | Ansible LXC (CT100) has no static IP — DHCP only | 🟡 Medium | Static DHCP reservation in OPNsense once production instance is live. | [`services/0001`](adr/services/0001-opnsense.md) |
| R-27 | `lib-synology-dsm` `verify_ssl=False` default — MitM risk in non-lab environments | 🟡 Medium | Documented in repo CLAUDE.md HARD RULES. Flip default once step-ca + `ca-trust` roll out (Layer 2). | [`security/0004-certificate-strategy`](adr/security/0004-certificate-strategy.md) |
| R-29 | **Vault security posture below ADR** (verified 2026-08-29): no audit device enabled (ADR: mandatory→Loki); no AppRole / per-service policies (auth = token+oidc; policies = admins/default/snapshot only); playbooks operate with the **root token**; unseal material (`vault-init.json`) on the controller disk (ADR: cold storage off-platform) | 🔴 Critical | Build items (roadmap, post-P2): enable file audit device → promtail→Loki; per-service policies + AppRole for rotation/CI; retire root-token use to break-glass only; move unseal keys off-platform. Until then compensations: Vault reachable only on SVC VLAN, VPN-only UI, raft-snapshot DR, secrets-validate. | [`security/0001`](adr/security/0001-secret-storage.md) |
| R-28 | Privileged-access **MFA not enforced** — `adm_*`, break-glass `{org}`, and interactive `svc-*` logins are key/password-only, no second factor | 🟡 Medium | Structural controls present (admin/user split, group RBAC, sudo-password, bastion session recording, VPN-only admin). **MFA on privileged logins via Authentik is the outstanding control** — effectively required by **NIS2 Art. 21(2)(j)** / **ISO 27001 A.8.5**; blocks privileged-access compliance sign-off. `identity/0004` CISO map already flags NIS2 "⚠ partial — MFA via Authentik future". | [`identity/0001`](adr/identity/0001-authentication.md), [`identity/0004-os-accounts`](adr/identity/0004-os-accounts.md) |

---

## A — Assumptions (Open)

| ID | Assumption | Impact if wrong |
|---|---|---|
| A-01 | `srv-proxmox-poc-01` hardware is the only production host until a second node is procured | Hardware HA impossible; all HA is software-only per [`services/0004 §Cluster placement`](adr/services/0004-database-strategy.md) |
| A-04 | Cloudflare is available as public DNS + DNS-01 ACME provider | Public cert issuance blocked — fallback = manual DNS-01 with alternative provider |
| A-13 | Nexus OSS (Apache 2.0) remains free for required formats | Forced migration to paid tier; version pinned per [`feedback_never_latest_docker`] rule |
| A-15 | Telenet provides a /27 static subnet on the WAN interface | OPNsense WAN config must match; verify before production deployment |

---

## I — Issues (Open)

| ID | Issue | Severity | Source |
|---|---|---|---|
| I-17 | Terraform scaffold incomplete — OPNsense production VM not deployed; top blocker | High | [`status.md §Top blockers`](status.md) |
| I-18 | NetBox not deployed — CMDB / IPAM source of truth unavailable | High | [`services/0003-netbox-cmdb`](adr/services/0003-netbox-cmdb.md) |
| I-11 | Ansible LXC (CT100) has no backup job and no `onboot=1` | High | [`infra/0008-backup-strategy`](adr/infra/0008-backup-strategy.md) |
| I-12 | Proxmox firewall disabled on prod and non-prod nodes | High | [`security/0003-hardening`](adr/security/0003-hardening.md) |
| I-13 | Synology NAS firewall disabled — all services open on `10.6.0.0/20` | Critical | [`security/0003`](adr/security/0003-hardening.md) |
| I-14 | No 2FA on any NAS human account | Critical | [`identity/0001-authentication`](adr/identity/0001-authentication.md) |
| I-15 | DSM 7.1.1-42962 outdated — unpatched CVEs likely | High | [`security/0003`](adr/security/0003-hardening.md) |
| I-16 | Python 2.7 (EOL) installed on NAS | High | [`security/0003`](adr/security/0003-hardening.md) |
| I-22 | Rune VM migration runbook missing | High | `platform-setup` pending runbook |
| I-23 | `FileStation.upload()` return dict doesn't match v1.0 contract | High | `lib-synology-dsm` audit pending |
| I-24 | `client.py` hardcoded 30s timeout — no per-op timeout | High | `lib-synology-dsm` audit pending |
| I-25 | Cloudflare API token rotation runbook missing | High | `platform-setup` pending runbook |

---

## D — Dependencies (Open)

| ID | Dependency | Required by | Risk if unavailable |
|---|---|---|---|
| D-01 | Second Proxmox host for hardware HA | Production promotion | Host/site failure unprotected |
| D-04 | ISP uplink (Telenet static /27) | OPNsense production | Public access blocked |
| D-05 | Cloudflare account + API token | Public DNS + ACME | Public cert issuance blocked |
| D-10 | Customer SMTP relay credentials | Layer 5 notifications | Platform email delivery fails in prod |
| D-12 | NAS backup target for Vault snapshots | Layer 2 sign-off | Vault snapshot restore impossible |
| D-13 | Authentik operational before Layer 5 OIDC wiring | Layer 3 → Layer 5 | OIDC config fails |
| D-14 | PostgreSQL HA cluster operational before GitLab/Authentik/NetBox deploy | Layer 5 | Service startup fails |
| D-15 | Nexus cache seeded before CI pipelines run at scale | Layer 5 | First pipelines hit internet, slow + brittle |

---

## Resolved / closed

Captured here for audit trail. Full original entries are preserved in the pre-refactor archive at [`docs/archive/2026-04-14-pre-cleanup/`](archive/2026-04-14-pre-cleanup/) where applicable.

| ID | Item | Resolved |
|---|---|---|
| R-21 | ADR number collision between `lib-synology-dsm` and `doc-platform-core` | 2026-04-14 — scoped refactor uses per-scope numbering, no global collision possible |
| R-22 | `vmbrMGMT` live link to Arista prod fabric VLAN 600 | 2026-04-03 — renamed `vmbrMGMT` → `vmbrFAB`, bridge disabled |
| R-23 | Nexus OSS VM provisioned at 2 GB RAM | 2026-04 — corrected to 6 GB in `infra-terraform-proxmox` |
| R-02 | PostgreSQL single instance | 2026-04-13 — superseded by [`services/0004-database-strategy`](adr/services/0004-database-strategy.md) (HA from day 1) |
| R-04, R-11 | Teleport-related risks | 2026-04 — Teleport removed from plan; OOB via `vmbrOOB` + break-glass per [`identity/0004 §5`](adr/identity/0004-os-accounts.md) |
| R-09, R-15 (Arista), R-16, R-17 | Arista fabric risks | 2026-04 — Arista fabric out of current scope; OPNsense is the platform firewall per [`services/0001`](adr/services/0001-opnsense.md) |
| R-13, R-24, R-25, R-26 | Licensing drift (Vault BUSL, Redis ≥7.4, AGPL tools) | 2026-04-14 — consolidated into [`security/0005-licensing-policy`](adr/security/0005-licensing-policy.md) |
| I-26 | NFS routing from PoC VM to NAS untested | 2026-04-01 — storage architecture corrected; VMs never mount NFS directly |
| I-27 | `vmbrOOB` break-glass bridge not created | 2026-04-03 — created via Proxmox API; `active=0`, `autostart=0` |
| I-28 | Flat ADR-0010 naming convention had stale prod examples | 2026-04-14 — folded into [`naming/0001-infra`](adr/naming/0001-infra.md) |
| I-29 | Flat ADR-0015 missing WireGuard specification | 2026-04-14 — folded into [`infra/0004-network-architecture`](adr/infra/0004-network-architecture.md) |
| I-30 | `ansible-platform` had no OPNsense role or bootstrap playbook | 2026-04-03 — role scaffold + bootstrap playbook created (ansible-platform PR #4) |
| I-19, I-20, I-21 | Stale GitHub issues, Terraform state backup, OOB gateway doc fixes | 2026-03-28/29 |
| I-01..I-06 | OpenClaw token, Discord stability, template extensions, npm/Go proxy choice | 2026-03-25 |

---

## Review cadence

| Layer (per [`status.md`](status.md)) | RAID review |
|---|---|
| Layer 1 in active work | Weekly |
| Layer 2 blocked (Vault/step-ca) | Weekly once started |
| Layer 3+ | Bi-weekly |
| Production promotion | Full RAID review required before go-live |

---

## Status legend

| Symbol | Meaning |
|---|---|
| 🔴 Critical | Immediate attention required |
| 🟡 Medium | Monitor and plan mitigation |
| 🟢 Low | Accept or defer |
