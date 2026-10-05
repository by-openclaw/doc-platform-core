# infra/0010 — Provisioning Standard: Ansible Only

**Status:** Proposed — awaiting owner approval
**Date:** 2026-10-05
**Scope:** How configuration reaches a host or a service of the platform: the roles and playbooks of `ansible-platform`. Not in scope: guest creation (`infra/0003-terraform-standard`), application source code, the checks that guard the repository itself.
**Related:** `services/0007-provisioning-orchestrator` (Ansible-first), `infra/0003-terraform-standard`, `security/0001-secret-storage`, `security/0003-hardening`, `OPERATING-STANDARD.md §5.3.1`

---

## Context

The owner's rule — given on 2026-09-27 and again on 2026-10-04 — is that the platform is provisioned by Ansible and by nothing else: no shell, bash, Python, Go or any other program in between. Until now that rule existed only as an instruction. No ADR states it, `OPERATING-STANDARD.md §5.3.1` describes how shell scripts should be written, and `services/0008` names a Python script as its proof of health. The compliance walk of the 32 services therefore had no check for it, and nothing in CI refused a script.

Measured on `ansible-platform` on 2026-10-05, after 45 conversions that day:

| What is not plain Ansible | Count |
|---|---|
| Tasks whose module is `shell` | 32 |
| An interpreter handed a program (`sh -c`, `bash -c`, `python -c`, `php -r`) in a task or in a scheduled job | 36 |
| Programs shipped by a role and run on a host (Python, Ruby, jq) | 14 |
| Tasks whose module is `raw` | 4 |

Why it matters, from what was found while converting:

- **Not reviewable as state.** A script says how; a module says what. Three scripts read the wrong column or the wrong field for months without anyone seeing it (the networkd guard, the container-scan probe that never ran, a job that replaced a list it meant to extend).
- **Not safe under `--check`.** A script runs or is skipped as a whole; it cannot report what it would change.
- **It leaks secrets.** A value given to a script through a command line or through the task environment is readable in the host's process list for as long as the task runs (proved on 2026-10-05). Tokens, unseal keys, AppRole secret ids and database passwords travelled that way.
- **It hides dependencies.** A helper program needs an interpreter and libraries on the host that no role declares.

## Decision

1. **One provisioning tool.** Every change to a platform host or service is made by a role or a playbook of `ansible-platform`. Terraform creates guests and nothing inside them.
2. **A task is a module call or one plain command.** A command is given as an argument list (`argv`), with its input on standard input when it needs one. Parsing, comparing and deciding happen in Ansible expressions.
3. **Forbidden in roles and playbooks:**
   - the `shell` module;
   - a program handed to an interpreter — `sh -c`, `bash -c`, `python -c`, `php -r`, `perl -e`, `ruby -e` — in a task, in an argument list or in a unit's command;
   - a program file shipped by a role and run on a host or in a container (`.sh`, `.py`, `.rb`, `.pl`, `.go`, `.jq`, `.php`);
   - the `raw` module, except to bootstrap a host that has no Python yet; each use is named with its reason.
4. **Scheduled work on a host** is a systemd unit and timer written by `roles/host_job`, whose commands are argument lists. No shell one-liner in a unit.
5. **Secrets never travel on a command line nor in a task environment.** They are module arguments, the standard input of a command, or a `0400` file in `/run` removed when the task ends.
6. **Service APIs are called with modules** (`uri` or a collection's module), delegated to a host that is allowed to reach the service.
7. **Not concerned:** an application's own configuration written in the language that application reads (GitLab's `gitlab.rb`, NetBox's `extra.py`) — each such file is named in the guard with its reason; Jinja expressions; the guard scripts under `ansible-platform/scripts/check_*.py`, which read the repository and provision nothing.

### Enforcement

- `scripts/check_no_embedded_scripts.py --ratchet` runs on every commit and in CI. It counts the four kinds above per file; a count may only go down. A new script, or one moved to another file, is refused.
- The counts are the progress number of the migration. Zero in all four is compliance.
- The per-service compliance checklist gains one check: the service's role and playbook have no entry in the guard's baseline.

### Exceptions

An exception is a file named in the guard with the reason it cannot be otherwise, approved by the owner in the pull request that adds it. There is no standing exception for convenience.

## Consequences

- `OPERATING-STANDARD.md §5.3.1` no longer describes platform provisioning. This proposal rewrites it as a pointer to this ADR (see the same pull request).
- Helper programs that talk to a service's API (JumpServer, GitLab RBAC, Discord, pgAdmin, the GRC registry) are rebuilt as role tasks. They are the slowest part of the migration.
- Some tasks become several: a loop that ran inside one script becomes one module call per item. Reports that took seconds can take a minute. That cost is accepted.
- Every conversion is proven before it is merged: the old and the new output compared on the platform, or a test that must fail when given a wrong input.

## Migration

| Milestone | Target |
|---|---|
| No `shell` task and no shell one-liner in a scheduled job | 2026-10-08 |
| No shipped program, no interpreter given a program, no unexplained `raw` | 2026-10-16 |
| Each service rebuilt from an empty guest by its playbook alone | after the two above, one service at a time |

The dates are estimates; the four counts are reported daily.

## Pending decisions (owner)

1. **Controller-side Python tools in `ansible-platform/scripts/`** that are not guards: `fw_apply_direct.py`, `fw_chain_phase1_apply.py`, `fw_prune_audit.py`, `fw_validate.py`, `fw_verify_health.py`, `opn_oob_admin.py`, `impacted_plays.py`, `deployment_vars.py`. Some act on the firewall. Proposed: each is classified — a repository check stays, anything that changes or verifies the platform becomes a playbook — and `services/0008` then names a playbook as its proof instead of `fw_verify_health.py`.
2. **`platform-setup`** (tool installers on the controller, the subject of the former §5.3.1): inside this rule or outside it.
3. **`lib-opnsense`** (the Python library the firewall catalog is applied with): confirmed as a library consumed by Ansible modules, not as a script.
