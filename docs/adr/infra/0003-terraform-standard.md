# infra/0003 — Terraform Standard

**Status:** Draft
**Date:** 2026-04-13 (supersedes flat ADR-0008, 2026-03-29)
**Scope:** Terraform state management, backend evolution, version pinning, and operational rules. Does not define individual resource schemas or module design.
**Related:** `git/0002-platform-strategy`, `security/0001-secret-storage`, `naming/0004-automation §9`, `infra/0002-platform-charter`

---

## Context

Terraform state (`terraform.tfstate`) is the source of truth for all managed infrastructure. Losing it means losing the ability to reconcile real infrastructure with config — forces destroy / recreate cycles and is a compliance-visible incident.

The platform currently runs Terraform from a single operator workstation. State is local on that workstation, which is an ephemeral VM. A remote backend is required to survive machine loss. A later migration to GitLab-managed state brings locking and multi-operator support, but that requires GitLab CE Layer 5 deployment first.

## Decision

### State backend evolution

| Phase | Backend | When |
|---|---|---|
| **Current** | Local file + **Synology NAS** backup | Pre-GitLab CE (now) |
| **Target** | **GitLab-managed HTTP backend** via `terraform { backend "http" { ... } }` | Once GitLab CE is deployed (Layer 5 per `infra/0002-platform-charter`) |

**Current (pre-GitLab) rules:**

1. State lives locally on the operator's workstation in `infra-terraform-*/environments/{env}/terraform.tfstate`.
2. State is backed up to Synology NAS at `/by-terraform-state/{env}/terraform.tfstate` **after every `apply` or `destroy`**.
3. Backup is triggered automatically by a wrapper script (`infra-terraform-*/scripts/tf.sh`) that wraps every state-mutating `terraform` invocation.
4. Manual sync is available via `scripts/backup-state.py --env {env}` — runbook content, not in this ADR.
5. **State is current before any session involving Terraform work closes.** Unsynced state is a compliance violation.
6. **No state locking in Phase 1.** Single operator only. Concurrent Terraform runs are forbidden — enforced by convention until GitLab backend provides native locking.

**Target (GitLab backend) rules:**

1. Backend config in `environments/{env}/backend.tf`:
   ```hcl
   terraform {
     backend "http" {
       address        = "https://gitlab.by-research.be/api/v4/projects/<id>/terraform/state/{env}"
       lock_address   = "https://gitlab.by-research.be/api/v4/projects/<id>/terraform/state/{env}/lock"
       unlock_address = "https://gitlab.by-research.be/api/v4/projects/<id>/terraform/state/{env}/lock"
       lock_method    = "POST"
       unlock_method  = "DELETE"
       retry_wait_min = 5
     }
   }
   ```
2. **Locking is automatic** — GitLab's Terraform state API enforces mutual exclusion per state key.
3. State access credentials come from Vault at `secret/{env}/terraform/gitlab-state-token` (per `security/0001-secret-storage`).
4. Synology NAS backup is retained for disaster recovery only — no longer the primary backend.
5. Migration is a one-time operation per state file: `terraform state pull` from local → `terraform state push` to the new GitLab backend, verified diff-free.

### Version and provider pinning

1. **Terraform version is pinned** per repo in `required_version`:
   ```hcl
   terraform {
     required_version = "~> 1.7"
   }
   ```
2. **All providers are pinned to a minor version** with `~>`:
   ```hcl
   required_providers {
     proxmox = {
       source  = "bpg/proxmox"
       version = "~> 0.60"
     }
   }
   ```
3. **`.terraform.lock.hcl` is committed to git**. Every PR runs `terraform init` and fails if the lock file would change, unless the PR explicitly upgrades a provider.
4. **Provider upgrades are PRs of their own** — never bundled with feature changes. The upgrade PR's description documents the `CHANGELOG` diff and any breaking changes.

### Formatting and static analysis

Every Terraform repo runs the following in CI before `plan`:

| Check | Tool | Gate |
|---|---|---|
| Format | `terraform fmt -check -recursive` | Hard fail on diff |
| Validate | `terraform validate` | Hard fail on error |
| Lint | `tflint` with platform-wide ruleset | Hard fail on error |
| Security / IaC scan | `checkov` | See `security/0003-hardening` Trivy/Checkov gate threshold (⚠ TBD) |
| Cost estimation (optional) | `infracost` | Advisory only, no gate |

CI runs `terraform plan` on every PR; `terraform apply` is a manual protected job that runs only after PR merge to `main`.

### Resource naming

Terraform resources, modules, and variables follow `naming/0004-automation §9`:

- Module path: `modules/{product}-{purpose}/`
- Module instance: `module "{purpose}_{index}" { source = "..." }`
- Resource `name` attribute: `{site}-{service}-{seq:02d}` per `naming/0001-infra`

Example:
```hcl
module "netbox_01" {
  source    = "./modules/proxmox-vm"
  site      = "vm"           # virtual pool on BR Proxmox
  service   = "nbox"         # NetBox short code
  seq       = "01"
  env       = "prod"         # written to NetBox custom field, not to name
  # → resulting VM name: vm-nbox-01
}
```

### State file security

1. **State files contain secrets** — Terraform embeds resource attributes including passwords, keys, and tokens. Treat state as secret-equivalent.
2. **Local state files on operator workstation** are stored under `~/repos/infra-terraform-*/environments/{env}/`. The workstation disk is encrypted at rest (LUKS). Backups to NAS go over OOB/MGMT network only.
3. **GitLab-managed state** is encrypted at rest by the GitLab server; access is scoped to project maintainers + specific CI job tokens.
4. **No state files in git.** `.gitignore` includes `*.tfstate`, `*.tfstate.backup`, `.terraform/`.
5. **State files never appear in logs, screenshots, or chat.** Redaction rule from `security/0001-secret-storage` applies.

## Consequences

- **State survives workstation loss** — restore from Synology NAS in Phase 1, restore from GitLab in Phase 2.
- **Migration to GitLab backend is a one-time operation** per state file; no code changes needed beyond the `backend` block.
- **State locking is deferred to Phase 2.** Concurrent Terraform runs are a process discipline issue until GitLab backend is live.
- **Version pinning prevents silent provider upgrades** from breaking `plan` output between runs.
- **CI gates catch drift and errors before apply** — format, validate, lint, security scan all run on every PR.
- **Resource naming is single-sourced** — Terraform doesn't invent its own convention, it follows `naming/0004-automation §9`.

## Revision triggers

Revise when:
- GitLab CE is deployed and state migrates to the GitLab backend
- Terraform is replaced by OpenTofu (or another tool) — would change backend configuration and version pinning syntax
- Multi-operator workflow is introduced before GitLab CE is deployed (forces an interim locking solution)
- Provider ecosystem changes — e.g. `bpg/proxmox` is archived and a replacement is adopted
- A new static analysis tool is adopted (e.g. `checkov` replaced by a different scanner)

## CISO mapping

| Framework | Controls covered |
|---|---|
| ISO 27001:2022 | A.8.13 (information backup — state backed up after every mutation), A.8.32 (change management — wrapper enforces backup on state-changing commands), A.8.6 (capacity management — NAS + local redundancy) |
| NIS2 | Art. 21(2)(c) (business continuity — state survives workstation loss, documented restore path) |
