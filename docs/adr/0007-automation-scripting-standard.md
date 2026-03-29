# ADR-0007 — Automation & Scripting Standard

**Status:** Accepted  
**Date:** 2026-03-29  
**Author:** @yboujraf + Rune  

---

## Context

All platform automation (shell scripts, Ansible roles, Terraform modules, Python tooling) must follow a consistent pattern. The reference implementation is `by-systems/odoo-install` — study it before contributing.

---

## Decision

### 1. Separation of Concerns

Each script does **one thing**. Orchestrators call scripts in order. No monoliths.

```
core-apply.sh      → installs base dependency
service-apply.sh   → configures the service
dns-sync-apply.sh  → manages DNS records
bootstrap-apply.sh → orchestrates: core → service → dns
destroy-all.sh     → full teardown in reverse order
```

### 2. Idempotency — Mandatory

Every script must be safe to run multiple times. Running apply twice = same result as running it once.

- Check before create: if resource exists and matches spec → no-op
- Check before delete: if resource absent → no-op, no error
- No side effects from repeated runs

### 3. Present / Absent Intent — Mandatory

Every script pair:

| Script | Intent |
|---|---|
| `<name>-apply.sh` | Ensure resource **present** — create or update to match spec |
| `<name>-clean.sh` | Ensure resource **absent** — remove cleanly, no data loss unless `--purge` |

`apply` = desired state enforced. `clean` = resource removed. Never ambiguous.

### 4. Dry-Run as Gate — Mandatory

Every apply script must support `--dry-run` (or `--check`):

```bash
bash service-apply.sh --dry-run
```

Dry-run behaviour:
- Validates all prerequisites (deps, versions, config files, spec compliance)
- Reports what **would** change — no writes, no mutations
- **If any check fails: exit non-zero. Apply is blocked until dry-run is clean.**
- CI pipelines run dry-run first. Apply only executes if dry-run exits 0.

This is not optional. A failed dry-run = hard stop.

### 5. Spec Validation — Mandatory

Before any apply, the script validates that the environment matches the declared spec:

- Required versions present (e.g. `nginx >= 1.24`, `python >= 3.11`)
- Required config files exist and parse correctly
- Required credentials/tokens present (from `credentials.yml` or env)
- Declared instances/resources are internally consistent

**If spec validation fails → no apply. Ever.**

### 6. Configuration — YAML, not flags

Runtime config lives in a YAML file (`instances.yml`, `credentials.yml`, etc.), not in command-line flags. Scripts read config; they do not take config as arguments.

```yaml
# instances.yml
instances:
  - name: odoo-prod
    enabled: true
    env: prod
    version: "17.0"
    ...
```

Flags are reserved for: `--dry-run`, `--purge`, `--status`, `--test`.

### 7. Documentation — Mandatory

Every script directory must contain a `<name>-report.md` (or `README.md`) with:

- **Structure** — what each script does, what it touches
- **Usage** — exact commands, flags, order of execution
- **Verification** — commands to confirm the desired state after apply
- **Rollback** — how to recover if something goes wrong
- **TODO** — known gaps, planned improvements

### 8. Logging & Output

- Use consistent prefixes: `[INFO]`, `[WARN]`, `[ERROR]`, `[DRY-RUN]`
- Dry-run output must be clearly prefixed: `[DRY-RUN] would create ...`
- No silent failures. Exit codes must be meaningful.

### 9. Collaborative Workflow

Scripts are co-authored: My Lord defines scope, intent, and acceptance criteria. Rune drafts, proposes, and iterates. Nothing is applied without review.

Workflow:
1. Define scope + tasks together
2. Write dry-run + spec validation first
3. Implement apply/clean
4. Test: dry-run clean → apply → verify → clean → verify absent
5. Document
6. Commit

---

## Consequences

- Every automation repo follows this pattern without exception
- PRs that break idempotency, remove dry-run, or skip documentation are rejected
- `odoo-install` is the canonical reference — read it before writing new automation

---

## Reference

- `by-systems/odoo-install` — reference implementation (proxy, cert, DNS, nginx, postgres)
- `by-openclaw/ansible-platform` — Ansible roles follow same present/absent pattern via `state:` variables
- `by-openclaw/infra-terraform-proxmox` — Terraform plan = dry-run gate, apply only after plan is clean
