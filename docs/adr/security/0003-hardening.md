# security/0003 — Platform Hardening Baseline

**Status:** Draft — several policy thresholds pending (`⚠ TBD` in §Pending decisions)
**Date:** 2026-04-13 (supersedes flat ADR-0021, 2026-04-02)
**Scope:** Platform-wide hardening baseline — non-root processes, image hygiene, patch cadence, scanning gates, audit logging.
**Related:** `security/0001-secret-storage`, `security/0004-certificate-strategy`, `identity/0004-os-accounts §5` (SSH break-glass), `git/0003-configuration`, `infra/0006-logging` (future)

---

## Context

The platform runs containerized services across multiple VMs. Without a formal hardening baseline, common failure modes are silently present: service containers running as root, dev tools in production images providing attacker footholds, plaintext secrets leaking into layers or env vars, undefined patch cadence, and missing container scan gates. ISO 27001:2022 A.8.8 and NIS2 Art. 21(2)(e) require vulnerability management evidence that only exists if the baseline is documented and enforced.

## Decision

### 1. Non-root processes — mandatory

Every container and service process **must** run as a non-root UID. No exceptions without explicit written approval documented in a per-service `docs/override-hardening.md`, reviewed annually.

**Rules:**
- Container images set a non-root `USER` directive before `ENTRYPOINT`
- Host services (systemd units) use dedicated system users, not `root`
- Init containers that need privileged capabilities drop them before the main process starts
- `docker run --user` override is not a workaround — the image itself must ship non-root

### 2. Image hygiene

**Container images must not include** `gcc`, `make`, `curl`, `wget`, `nc`, `nmap`, or any build toolchain unless operationally required for runtime functionality.

**Rules:**
- Base image is a slim or distroless variant whenever upstream offers one
- Multi-stage Dockerfiles — build tools in the `build` stage, runtime image has only what the process needs
- Exceptions documented in the service's `docs/hardening.md` with justification and risk statement
- `CI` lint rule (Trivy + Dockerfile analysis) catches common violations

### 3. No plaintext secrets in images or env vars

Secrets are **injected at runtime** via Vault Agent, K8S secrets, or equivalent — never baked into the image or passed as `docker run -e SECRET=...`.

**Rules:**
- No `ARG SECRET` or `ENV SECRET` in any Dockerfile
- No `.env` files baked into `/etc/` or application dirs
- Vault Agent sidecar template pattern is documented in `platform-setup/tools/*/docs/hardening.md` per service
- Runtime env vars that reference Vault paths (not values) are allowed

See `security/0001-secret-storage` for Vault path conventions.

### 4. Container scanning — Trivy in CI

Trivy runs in every repo that builds a container image. A CI gate blocks image promotion if the severity threshold is exceeded.

**Gate threshold:** ⚠ **TBD** — see §Pending decisions

**Rules:**
- Trivy runs on every PR against the main branch
- Scan targets: OS packages, language dependencies, config misconfigurations, and secrets
- Scan results are archived per build for evidence
- Suppression (`.trivyignore`) requires a documented justification per suppression entry

### 5. Host hardening — Lynis

Lynis runs on every host VM (Linux) as part of the Ansible `hardening` role. Minimum hardening score is enforced.

**Lynis score target:** ⚠ **TBD** — see §Pending decisions

**Rules:**
- Lynis runs as part of the hardening Ansible role (`ansible-platform/roles/hardening`)
- Report is collected post-run and archived
- Score below threshold fails the playbook

### 6. Patch cadence

Critical and high-severity CVEs must be patched within defined windows from public disclosure.

**Patch windows:** ⚠ **TBD** — see §Pending decisions

**Rules:**
- Patching is automated where possible (Ansible `hardening` role, unattended-upgrades, container image rebuild)
- Deviations from cadence must be documented in a per-service exception with expiry date
- Incident response activates if a critical CVE cannot be patched within window (see NIS2 Art. 21(2)(b) — incident handling)

### 7. Filesystem policy

Container root filesystems are read-only where feasible; writable paths use `tmpfs` or explicit volume mounts.

**Filesystem mode:** ⚠ **TBD** — see §Pending decisions

**Rules (once policy is defined):**
- `docker run --read-only` or equivalent k8s SecurityContext
- Writable paths declared explicitly per service
- Persistent state uses named volumes, never bind mounts into container paths

### 8. Audit logging

All administrative actions are logged. Log destination is Loki (see `infra/0006-logging` when refactored).

**Rules:**
- Host-level auditd enabled on all VMs, forwarded to Loki via promtail
- SSH session activity logged via sshd + PAM, forwarded to Loki
- Bastion / jump host session recording via Teleport (future — see revision triggers)
- Audit log retention: minimum 90 days, matching NIS2 Art. 23 incident notification windows
- Audit logs themselves are immutable from the perspective of platform service accounts — write-once to Loki

### 9. SSH hardening

SSH is hardened per `identity/0004-os-accounts §5` — password authentication is disabled globally except for the `break-glass` group from the OOB CIDR. Everything else is key-only.

**Key rules (summarized here, authoritative in identity/0004):**
- `PasswordAuthentication no` by default
- `PubkeyAuthentication yes`
- Only ED25519 keys, passphrase mandatory (no empty-passphrase keys)
- `PermitRootLogin no` everywhere except break-glass via `Match Group`
- Non-standard port (`22222`) to reduce drive-by scan noise
- fail2ban enabled, OOB CIDR whitelisted

### 10. Cross-OS note

Current baseline targets **Linux** hosts provisioned via SSH. When **Windows 11 / Windows Server** hosts are introduced (planned, WinRM transport — see `git/0003-configuration §11`), this ADR is amended to add the Windows equivalents: local security policy (LGPO / STIG), Windows Defender ATP, Windows Update for Business patch cadence, and equivalent container rules for Windows containers.

Until then, Windows hosts are out of scope of this baseline.

## Pending decisions

The following policy thresholds must be set by @yboujraf before this ADR moves from Draft to Accepted.

| # | Decision | Options / reference | Status |
|---|---|---|---|
| 1 | **Patch cadence — critical CVE** | e.g. `critical ≤ 72h`, `high ≤ 7d`, `medium ≤ 30d` | ⚠ TBD |
| 2 | **Lynis minimum score** | Typical values: `≥ 70` (baseline), `≥ 80` (strong), `≥ 90` (strict) | ⚠ TBD |
| 3 | **Trivy CVE gate threshold** | Typical values: `zero critical`, `zero critical + zero high`, `zero critical + ≤5 high with SLA` | ⚠ TBD |
| 4 | **Filesystem policy — read-only root?** | `read-only root, tmpfs for /tmp /run` vs `writable with no-new-privileges only` | ⚠ TBD |
| 5 | **Audit log retention** | Current target `≥ 90 days` (NIS2 minimum) — confirm or extend | ⚠ Proposed |
| 6 | **Lynis scope** | Linux hosts only, or include container base images via `lynis audit system --profile ...`? | ⚠ TBD |

**Rule:** until each decision is locked, the corresponding section above carries the `⚠ TBD` marker. Once a decision is made, this ADR is amended and the marker removed. Partial decisions are better than none — do not block the entire ADR on one unresolved threshold.

## Consequences

- **Formal vulnerability management evidence** — ISO 27001:2022 A.8.8 and NIS2 Art. 21(2)(e) are directly supported by Trivy + Lynis + patch cadence
- **Non-root everywhere** — container escape blast radius is drastically reduced
- **CI gate prevents unpatched images reaching production** — images are blocked at build time, not caught by runtime scanning
- **Secret leakage via image layers is eliminated** — no plaintext secrets in Dockerfiles or env vars
- **Baseline is uniform across services** — per-service `docs/hardening.md` extends with service-specific additions, but never relaxes the baseline
- **Windows is deferred** — cross-OS hardening is amended when WinRM hosts are introduced
- **Known gap until §Pending decisions are locked:** several policy thresholds are placeholders. Incident response SLA cannot be fully automated until cadence + Lynis + Trivy thresholds are set.

## Revision triggers

Revise when:
- Any `⚠ TBD` in §Pending decisions is resolved
- Trivy is replaced by a different scanner (Snyk, Grype, etc.)
- Lynis is replaced or a Windows equivalent is added
- A new CVE severity level or scoring system supersedes CVSS 3.x
- Teleport is deployed for session recording (adds §8 details)
- Wazuh is deployed for host-based intrusion detection (adds §8 details)
- Windows 11 / Windows Server hosts are introduced (adds §10 Windows baseline)
- A new compliance framework (DORA, SOC 2, PCI-DSS) forces a threshold change

## CISO mapping

| Framework | Controls covered |
|---|---|
| ISO 27001:2022 | A.8.8 (technical vulnerabilities — ⚠ partial, thresholds TBD), A.8.9 (configuration management), A.8.15 (logging), A.8.25 (secure development lifecycle — ⚠ partial, Trivy gate TBD) |
| NIS2 | Art. 21(2)(e) (network and information systems security — ⚠ partial), Art. 21(2)(h) (basic cyber hygiene) |
| GDPR | Art. 32(1)(b) (ongoing confidentiality, integrity, availability — ⚠ partial) |
