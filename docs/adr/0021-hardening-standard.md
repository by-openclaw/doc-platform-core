<!--
  SCOPE GUARD — INFRA ADR
  ========================
  This template is for infrastructure decisions only.
  DO NOT use this template for:
  - Library implementation details (DI, test strategy, log format)
  - Developer tooling (devcontainer, pre-commit hooks)
  - App-level credential handling
  Wrong template = PR blocked.
-->

# ADR-0021: Platform hardening baseline

**Status:** Draft — thresholds defined per-tool during deployment phase
**Date:** 2026-04-02
**Deciders:** @yboujraf

---

## Context

The platform runs containerised services across multiple VMs. Without a formal hardening baseline:

- Service containers may run as root, widening the blast radius of any container escape
- Development tools (curl, gcc, wget) may be present in production images, providing attacker footholds
- Plaintext secrets may leak into container layers, environment variables, or logs
- Patch cadence is undefined — critical CVEs may go unaddressed for an unacceptable period
- ISO 27001:2022 A.8.8 and NIS2 Art. 21(2)(e) require vulnerability management and security controls

This ADR defines the platform hardening baseline. Per-service hardening configuration is documented in each tool's `docs/hardening.md`. The governing standard is `docs/standards/hardening-standard.md`.

## Decision

**Non-root processes are mandatory.** Every container and service process must run as a non-root UID. No exceptions without explicit approval documented in `docs/override-hardening.md`.

**Filesystem policy:** [OWNER TO DEFINE: read-only root filesystem where feasible? tmpfs for writable paths?]

**Development tool policy:** Container images MUST NOT include `gcc`, `make`, `curl`, `wget` unless operationally required. Exceptions must be documented in `docs/hardening.md`.

**No plaintext secrets in images.** Secrets are injected at runtime via Vault Agent or K8S secrets.

**Audit logging** is mandatory for all administrative actions. Teleport session recording applies to all bastion access.

**Patch cadence:** [OWNER TO DEFINE: e.g., critical CVEs within 72h, high within 7d, medium within 30d]

**Lynis score target:** [OWNER TO DEFINE: minimum Lynis hardening score for host VMs]

**Container scanning** (Trivy) runs in CI before any image is promoted. Pipeline gate blocks promotion if critical CVEs exceed the threshold: [OWNER TO DEFINE: threshold — e.g., zero critical / zero high unpatched beyond SLA]

## VM / Resource Spec

Not applicable — hardening standard applies across all VMs.

## Network

Not applicable — hardening standard is process/filesystem level, not network topology.

## Storage

Not applicable.

## TLS / PKI

Not applicable.

## CISO mapping

> Applies only to controls directly relevant to this ADR's scope.

### ISO/IEC 27001:2022

| Control | Title | Status | Notes |
|---|---|---|---|
| A.8.8 | Management of technical vulnerabilities | ⚠ Partial | Patch cadence defined as placeholder; Trivy in CI not yet gated |
| A.8.9 | Configuration management | ⚠ Partial | Non-root policy defined; per-service hardening docs pending |
| A.8.15 | Logging | ✓ Covered | Audit logging requirement defined; Teleport session recording in scope |
| A.8.25 | Secure development lifecycle | ⚠ Partial | Trivy scanning required in CI; gate threshold pending @yboujraf |

### NIS2 (Directive 2022/2555)

| Article | Requirement | Status | Notes |
|---|---|---|---|
| Art. 21(2)(e) | Security in network and information systems acquisition, development and maintenance | ⚠ Partial | Container scanning defined; patch SLA thresholds pending |
| Art. 21(2)(h) | Basic cyber hygiene practices and cybersecurity training | ⚠ Partial | Hardening baseline defined; training programme not yet scoped |

### GDPR (Regulation 2016/679)

| Article | Requirement | Status | Notes |
|---|---|---|---|
| Art. 32(1)(b) | Ability to ensure ongoing confidentiality, integrity, availability | ⚠ Partial | Hardening baseline addresses confidentiality; availability covered by monitoring ADR |

## Licensing

| Tool / Service | SPDX license | Tier | License ref |
|---|---|---|---|
| Trivy | Apache-2.0 | Free | https://github.com/aquasecurity/trivy/blob/main/LICENSE |
| Lynis | GPL-3.0 | Free | https://github.com/CISOfy/lynis/blob/master/LICENSE |

## Consequences

**Enables:**
- Formal ISO 27001:2022 A.8.8 and NIS2 vulnerability management evidence
- Consistent hardening baseline across all services
- CI gate prevents unpatched images from reaching PoC or Prod

**Constrains:**
- All services must run as non-root — images that default to root must be reconfigured
- CI pipeline must include Trivy gate before this standard is enforced end-to-end
- Per-service `docs/hardening.md` must be completed before a service is production-ready

**Known risks:**
- Patch cadence thresholds are placeholders — incident response window is undefined until @yboujraf sets values
- Lynis score target not yet defined — host hardening cannot be objectively assessed
- Container scanning gate not yet active in CI — critical CVEs may currently reach PoC undetected

## References

- `docs/standards/hardening-standard.md` — platform hardening standard (governing document)
- [CIS Docker Benchmark](https://www.cisecurity.org/benchmark/docker)
- [Trivy documentation](https://aquasecurity.github.io/trivy/)
- [Lynis documentation](https://cisofy.com/lynis/)
- [ISO 27001:2022 A.8.8](https://www.iso.org/standard/27001)
- [NIS2 Art. 21(2)(e)](https://eur-lex.europa.eu/legal-content/EN/TXT/?uri=CELEX%3A32022L2555)
