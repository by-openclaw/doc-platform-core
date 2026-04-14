# Platform Status

> **Last updated:** 2026-04-14
> **Maintained by:** Rune — update after any release, layer change, or RAID update
> **Reading order:** `status.md` → `roadmap.md` → [`docs/adr/README.md`](adr/README.md) → relevant repo `CLAUDE.md`

---

## Refactor milestone — 2026-04-14

The scoped ADR refactor is **complete**. 30 scoped ADRs across 7 scopes under [`docs/adr/`](adr/) replace the 34 pre-refactor flat ADRs. Loose documentation files that predated the refactor are archived under [`docs/archive/2026-04-14-pre-cleanup/`](archive/2026-04-14-pre-cleanup/). Top-level repo metadata (`README.md`, `CLAUDE.md`, `AGENTS.md`, `CONTRIBUTING.md`, `RAID.md`) is refreshed and consistent with the scoped structure.

This `status.md` tracks platform **deployment and delivery state** — not the documentation refactor, which is done.

---

## Layer progress

Per [`infra/0002-platform-charter §Layer model`](adr/infra/0002-platform-charter.md). Each layer must be documented, tested, and signed off before the next begins.

| Layer | Name | Status | Notes |
|---|---|---|---|
| 0 | Standards & Templates | ✅ **Complete** | 30 scoped ADRs, `OPERATING-STANDARD.md` current. Refactor completed 2026-04-14. |
| 1 | Proxmox Base | ⚠ **Partial** | `srv-proxmox-poc-01` operational. `ansible-platform/roles/hardening` live (sshd, fail2ban, ufw, postfix). Cloud-init template + `user-mgmt` role ⚠ pending verify. |
| 2 | Vault + step-ca | ❌ Not started | Blocker for every downstream layer. `ansible-platform/roles/ca-trust` and Vault policies pending. |
| 3 | Identity (Authentik) | ❌ Not started | Blocked on Layer 2. |
| 4 | Storage (Synology + MinIO) | ⚠ **Partial** | Synology DS1513+ operational, used for Terraform state backup per [`infra/0003-terraform-standard §State backend evolution`](adr/infra/0003-terraform-standard.md). MinIO not yet deployed. `lib-synology-dsm` exists and ships FileStation manager. |
| 5 | Platform Services | ❌ Not started | NetBox, GitLab CE, Vaultwarden, observability stack, Traefik — all blocked on Layers 2–4. OPNsense is the only Layer 5–adjacent service with a live test instance (see below). |

---

## Per-repo status

| Repo | Role | State |
|---|---|---|
| `doc-platform-core` | Platform documentation | ✅ Post-refactor clean — 30 scoped ADRs, 3 active `docs/` loose files, archives in place |
| `lib-opnsense` | Python library wrapping OPNsense API | ✅ `v1.0.0` — 54 managers, 1178 unit tests, 294 integration tests, bcrypt password verification, composite match keys, `AmbiguousMatchError`, field validators, structlog with Loki JSON output. Matches [`lib/python/0001-design-standard`](adr/lib/python/0001-design-standard.md). |
| `ansible-opnsense` | Ansible collection wrapping `lib-opnsense` | ✅ 54 modules (one per `lib-opnsense` manager). CRUD + error-path test playbooks per domain. Verbosity logging via `-v` / `-vv`. |
| `ansible-platform` | Ansible roles for platform-wide OS config | ⚠ Partial — `hardening` role live (sshd template with OOB break-glass per [`identity/0004-os-accounts §5`](adr/identity/0004-os-accounts.md)). `user-mgmt`, `git-config`, `key-mgmt`, `ca-trust`, `observability-client` roles ⚠ pending verify / pending write. |
| `infra-terraform-proxmox` | Terraform modules for Proxmox provisioning | ⚠ Partial — `vm-opnsense` module exists. State backed up to Synology NAS after every `apply`/`destroy` per [`infra/0003-terraform-standard`](adr/infra/0003-terraform-standard.md). OPNsense production VM ⚠ not yet deployed (test instance exists, see Top Blockers). |
| `lib-synology-dsm` | Python library wrapping Synology DSM API | ✅ FileStation + SystemManager + SharePermissionManager. Exact version and `lib/python/0001-design-standard` compliance status ⚠ pending audit. |
| `platform-setup` | Per-tool configs, runbooks, security hardening | ⚠ Partial — runbooks `opnsense/bootstrap.md` and `mailcow/lifecycle.md` ⚠ pending write (referenced from `services/0001-opnsense` and `services/0002-email-infrastructure` respectively). |

---

## OPNsense — the one Layer 5 service with a live instance

- **Test instance:** `vm-opnsense-01` on test VLANs (`10.11.x.x`), running OPNsense 26.1.5
- **Production instance:** ⚠ not yet deployed — tracked as top blocker
- **`svc-rune` API key:** stored in Vault once Vault is deployed; currently in `infra/secrets/` per the transitional JSON state noted in [`security/0001-secret-storage §Transitional state`](adr/security/0001-secret-storage.md)
- **Integration coverage:** live probe against 200/200 endpoints per earlier session work (2026-04-05). Used for `ansible-opnsense` integration test playbooks.

---

## Top blockers

Per [`docs/raid.md`](raid.md) and the scoped ADRs' `§Pending decisions` / `§Deferred decisions` sections.

| # | Item | Blocks | Source |
|---|---|---|---|
| 1 | **OPNsense production deployment** (only test instance exists today) | All Layer 2+ work — OPNsense is the platform firewall/DHCP/DNS/VPN router | [`services/0001-opnsense`](adr/services/0001-opnsense.md) + [`infra/0004-network-architecture`](adr/infra/0004-network-architecture.md) |
| 2 | **HashiCorp Vault deployment** | Layer 2 complete → unblocks Layers 3, 4, 5 | [`security/0001-secret-storage`](adr/security/0001-secret-storage.md) + [`infra/0002-platform-charter §Layer 2`](adr/infra/0002-platform-charter.md) |
| 3 | **step-ca deployment + `ca-trust` Ansible role** | All internal TLS services — missing root CA distribution silently breaks every internal HTTPS client | [`security/0004-certificate-strategy §Root CA distribution`](adr/security/0004-certificate-strategy.md) |
| 4 | **11 `⚠ TBD` thresholds** in `infra/0008-backup-strategy` | Backup policy cannot be fully automated until thresholds are locked (off-site target, schedules, retention, RTO, drill cadence, KMS) | [`infra/0008-backup-strategy §Pending decisions`](adr/infra/0008-backup-strategy.md) |
| 5 | **6 `⚠ TBD` thresholds** in `security/0003-hardening` | Patch cadence, Lynis score, Trivy CVE gate, filesystem policy, retention, Lynis scope | [`security/0003-hardening §Pending decisions`](adr/security/0003-hardening.md) |
| 6 | **4 deferred decisions** in `services/0004-database-strategy` | PostgreSQL pooler choice (`pgbouncer` vs `pgpool-II`), Patroni DCS (`etcd` vs `Consul`), Sentinel colocation, Redis replica count | [`services/0004-database-strategy §Deferred decisions`](adr/services/0004-database-strategy.md) |

---

## Recent updates

| Date | What changed |
|---|---|
| 2026-04-14 | **Refactor complete.** Loose `docs/` files archived (PR #38). Top-level repo metadata refreshed (PR #40) — `SOUL.md` + `USER.md` deleted as workspace duplicates per `OPERATING-STANDARD.md §3.2`. `docs/status.md`, `roadmap.md`, `raid.md` refreshed (this PR). |
| 2026-04-14 | Scoped ADR refactor merged — the final 12 PRs (#13 through #36) split 34 flat ADRs into 30 scoped ADRs across 7 scopes (identity / git / naming / security / infra / services / lib-python). `OPERATING-STANDARD.md §4.4` added (PR Content Standard). `OPERATING-STANDARD.md §§5.3.1–5.3.6` added (script-writing patterns from archived flat ADR-0007). |
| 2026-04-13 | Identity / git / naming / security / infra / services / lib refactors merged. `poc` dropped as env tier — 6 tiers only (`dev` / `test` / `staging` / `acc` / `prod` / `drp`). HA database clusters from day 1 per [`services/0004-database-strategy`](adr/services/0004-database-strategy.md). |
| 2026-04-12 | Repos moved from `~/.openclaw/workspace/repos/` to `~/repos/`. Workspace git uncommitted count went from ~420 to 0. |
| 2026-04-11 | `ansible-opnsense` — 54 modules implemented, one per `lib-opnsense` manager. Enum validators fixed against OPNsense 26.1.5 API probe (14 enum mismatches corrected). bcrypt password verification in `DiffEngine`. Integration test playbooks split per domain. |
| 2026-04-10 | `lib-opnsense v1.0.0` — 54 managers, 1178 unit tests, 294 integration tests. Composite match keys + `AmbiguousMatchError`. `try/except/log/raise` on every method that can throw. structlog with Loki JSON output. |
| 2026-04-05 | `lib-opnsense` 200/200 live probe coverage against OPNsense 26.1.5 (test instance). Field validators extracted from MVC model definitions. |

---

## How to update

After any release or layer change:

1. Update the relevant row in §Per-repo status
2. Update §Layer progress if a layer state changes
3. Update §Top blockers — add or remove as RAID / pending decisions change
4. Add a row to §Recent updates (keep newest first)
5. Commit: `docs: update platform status to YYYY-MM-DD`
6. Cross-update `docs/raid.md` if a blocker becomes resolved or a new risk appears
