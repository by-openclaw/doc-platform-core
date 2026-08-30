# ADR-0008 — Terraform State Management

**Status:** Accepted
**Date:** 2026-03-29
**Author:** @yboujraf + Rune

---

## Context

Terraform state (`terraform.tfstate`) is the source of truth for all managed infrastructure.
Losing it means losing the ability to reconcile real infrastructure with config — forces destroy/recreate cycles.

During PoC phase, state is stored locally on the Rune VM (`~/repos/infra-terraform-proxmox/environments/poc/`).
The Rune VM is ephemeral by nature (PoC, not HA). A remote backend is required to survive VM loss.

---

## Decision

### Phase 1 — PoC (current)

State is backed up to **Synology NAS** after every `apply` or `destroy`:

- **NAS path:** `/by-terraform-state/<env>/terraform.tfstate`
- **Backup script:** `infra-terraform-proxmox/scripts/backup-state.py`
- **Wrapper:** `infra-terraform-proxmox/scripts/tf.sh` — replaces bare `terraform` invocations

Backup is triggered automatically by `tf.sh` on successful `apply` or `destroy`.
Manual sync available at any time: `python3 scripts/backup-state.py --env poc`

**Rune also ensures state is current before ending any session involving Terraform work.**

### Phase 2 — GitLab CE (planned)

Migrate to **GitLab-managed Terraform state** when GitLab CE is deployed (Phase 5 in the platform rollout):
- Backend: `terraform { backend "http" { ... } }` pointing to GitLab state API
- Locking: built-in via GitLab
- No more NAS backup required for state (NAS backup kept for DR only)

---

## Backup procedure — Rune AI agent

The Rune agent (`claude-sonnet-4-6`) is responsible for:

1. Running `scripts/backup-state.py` after every Terraform session
2. Verifying the NAS contains a recent copy before closing any infra session
3. Restoring state from NAS if the Rune VM is rebuilt:
   ```python
   from synology_dsm.client import DSMClient
   from synology_dsm.filestation import FileStationManager
   c = DSMClient("10.6.224.6", port=5001, https=True, verify_ssl=False)
   c.login("rune-api", "<REDACTED:password>")
   FileStationManager(c).download(
       "/by-terraform-state/poc/terraform.tfstate",
       "environments/poc/terraform.tfstate"
   )
   ```

---

## Tooling

| Tool | Purpose |
|---|---|
| `scripts/tf.sh` | Terraform wrapper — auto-backup after apply/destroy |
| `scripts/backup-state.py` | Standalone backup script |
| `lib-synology-dsm` v0.6.1+ | FileStation upload/download API (`by-openclaw/lib-synology-dsm`) |

---

## Auth quirk (FileStation API — DS1513+ DSM 7.x)

Discovered via browser DevTools — documented here to avoid re-investigation:

- `SynoToken` must be passed in the **URL query string** (`?SynoToken=...`)
- Session must be passed as **cookie** `id=` (not form field `_sid`)
- Upload field is `path`, not `dest_folder_path`
- Login must use `session=DSM` (not `FileStation`)

This is handled transparently by `lib-synology-dsm`. Do not bypass the lib.

---

## Consequences

- ✅ State survives Rune VM loss — restore from NAS in < 1 minute
- ✅ Zero config change required for Phase 2 migration (just swap backend block)
- ⚠️ NAS is not HA — if NAS is unavailable, state is still safe on Rune VM (local)
- ⚠️ No state locking in Phase 1 — single operator only (acceptable for PoC)

---

## CISO mapping

> Applies only to controls directly relevant to this ADR's scope.

### ISO/IEC 27001:2022

| Control | Title | Status | Notes |
|---|---|---|---|
| A.8.13 | Information backup | ✓ Covered | Terraform state is backed up to Synology NAS after every apply/destroy |
| A.8.6 | Capacity management | ⚠ Partial | Local + NAS copies reduce loss risk, but NAS is not HA |
| A.8.32 | Change management | ✓ Covered | `tf.sh` wrapper enforces backup on state-changing commands |

### NIS2 (Directive 2022/2555)

| Article | Requirement | Status | Notes |
|---|---|---|---|
| Art. 21(2)(c) | Business continuity and backup management | ✓ Covered | State survives Rune VM loss; restore path is documented |

### GDPR (Regulation 2016/679)

Not applicable — this ADR covers infrastructure state, not personal data processing.

**References:**
- [infra-terraform-proxmox/scripts/](https://github.com/by-openclaw/infra-terraform-proxmox/tree/main/scripts)
- [lib-synology-dsm v0.6.1](https://github.com/by-openclaw/lib-synology-dsm/releases/tag/v0.6.1)
- VCS & CI/CD strategy decision document
- Platform charter document
