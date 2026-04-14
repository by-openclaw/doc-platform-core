# security/0003 — Platform Hardening Baseline

**Status:** Accepted
**Date:** 2026-04-14 (thresholds locked; supersedes flat ADR-0021, 2026-04-02)
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

**Gate threshold:** **Zero critical + zero high.** Any critical or high CVE fails the build. Medium and low are reported but non-blocking.

**Rules:**
- Trivy runs on every PR against the main branch
- Scan targets: OS packages, language dependencies, config misconfigurations, and secrets
- Scan results are archived per build for evidence
- Suppression (`.trivyignore`) requires a documented justification **per suppression entry** (CVE ID, reason, expiry date)
- Expired suppressions re-fail the gate automatically on next run

### 5. Host hardening — Lynis

Lynis runs on every host VM (Linux) as part of the Ansible `hardening` role. Minimum hardening score is enforced.

**Lynis score target:** **≥ 80** (strong baseline). Score below 80 fails the Ansible playbook.

**Rules:**
- Lynis runs as part of the hardening Ansible role (`ansible-platform/roles/hardening`)
- Report is collected post-run and archived
- Score below 80 fails the playbook — the host is not considered hardened
- **Scope:** Linux hosts only. Container images are covered by Trivy (§4), not Lynis — no overlap
- Lynis runs again on every `ansible-platform` apply, not just initial provisioning

### 6. Patch cadence

Critical and high-severity CVEs must be patched within defined windows from public disclosure.

**Patch windows (from public CVE disclosure):**

| Severity | Window |
|---|---|
| Critical | ≤ 72 hours |
| High | ≤ 7 days |
| Medium | ≤ 30 days |
| Low | ≤ 90 days |

**Rules:**
- Patching is automated where possible (Ansible `hardening` role, unattended-upgrades, container image rebuild)
- Deviations from cadence must be documented in a per-service exception with expiry date
- A critical CVE that cannot be patched within 72h activates incident response per NIS2 Art. 21(2)(b)
- Patch status is tracked by Trivy (§4) on next CI run — no separate tracker needed

### 7. Filesystem policy

Container root filesystems are **read-only by default**. Writable paths use `tmpfs` or explicit named volumes.

**Filesystem mode:** **Read-only root + `tmpfs` for `/tmp`, `/run`, `/var/run`.** Any additional writable path must be declared explicitly per service.

**Rules:**
- `docker run --read-only` or equivalent Kubernetes `SecurityContext.readOnlyRootFilesystem: true`
- `tmpfs` mounts for `/tmp`, `/run`, `/var/run` are added by default via the platform Compose/Helm templates
- Additional writable paths must be declared per service and justified in the service's `docs/hardening.md`
- Persistent state uses named volumes, never bind mounts into container paths
- Exceptions (writable root) are written to the per-service `docs/override-hardening.md` and reviewed annually (same rule as §1)

### 8. Audit logging

All administrative actions are logged. Log destination is Loki (see `infra/0006-logging` when refactored).

**Rules:**
- Host-level auditd enabled on all VMs, forwarded to Loki via promtail
- SSH session activity logged via sshd + PAM, forwarded to Loki
- Bastion / jump host session recording via Teleport (future — see revision triggers)
- Audit log retention: **180 days** (doubles the NIS2 Art. 23 minimum; matches ISO 27001 auditor expectations)
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

## Locked thresholds (history)

All six thresholds were locked on 2026-04-14, moving this ADR from Draft to Accepted.

| # | Decision | Locked value | Locked on |
|---|---|---|---|
| 1 | Patch cadence | critical ≤ 72h · high ≤ 7d · medium ≤ 30d · low ≤ 90d | 2026-04-14 |
| 2 | Lynis minimum score | ≥ 80 | 2026-04-14 |
| 3 | Trivy CVE gate | zero critical + zero high (per-entry `.trivyignore` with expiry) | 2026-04-14 |
| 4 | Filesystem policy | read-only root + tmpfs for `/tmp`, `/run`, `/var/run` | 2026-04-14 |
| 5 | Audit log retention | 180 days | 2026-04-14 |
| 6 | Lynis scope | Linux hosts only (containers via Trivy §4) | 2026-04-14 |

Any future change to a locked value requires a new ADR revision per §Revision triggers.

## Consequences

- **Formal vulnerability management evidence** — ISO 27001:2022 A.8.8 and NIS2 Art. 21(2)(e) are directly supported by Trivy + Lynis + patch cadence
- **Non-root everywhere** — container escape blast radius is drastically reduced
- **CI gate prevents unpatched images reaching production** — images are blocked at build time, not caught by runtime scanning
- **Secret leakage via image layers is eliminated** — no plaintext secrets in Dockerfiles or env vars
- **Baseline is uniform across services** — per-service `docs/hardening.md` extends with service-specific additions, but never relaxes the baseline
- **Windows is deferred** — cross-OS hardening is amended when WinRM hosts are introduced
- **Incident response SLA is fully automatable** — the patch cadence + Trivy gate + Lynis threshold define a measurable window from CVE disclosure to enforced fix

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
| ISO 27001:2022 | A.8.8 (technical vulnerabilities), A.8.9 (configuration management), A.8.15 (logging), A.8.25 (secure development lifecycle) |
| NIS2 | Art. 21(2)(e) (network and information systems security), Art. 21(2)(h) (basic cyber hygiene) |
| GDPR | Art. 32(1)(b) (ongoing confidentiality, integrity, availability) |
