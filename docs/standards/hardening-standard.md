<!--
| Field        | Value            |
|--------------|------------------|
| Created      | 2026-04-02       |
| Last updated | 2026-04-02       |
| Updated by   | Opus             |
-->

# Hardening Standard

**Status:** Draft
**Date:** 2026-04-02
**Owner:** @yboujraf
**ADR:** ADR-0021 (stub — pending @yboujraf sign-off)

## Rule

Every platform service must run as a non-root process, follow the filesystem security policy, store no plaintext secrets in the container or VM, enable audit logging, and be patched within the defined patch cadence.

## Requirements

1. Services MUST NOT run as root. All containers and service processes must use a non-root UID.
2. Container images MUST NOT include development tools (`gcc`, `make`, `curl`, `wget`) unless required for operation — document exceptions explicitly.
3. Filesystem policy: [OWNER TO DEFINE: read-only root filesystem where possible? tmpfs for writable dirs?].
4. No plaintext secrets in container images, environment variables visible to other processes, or config files committed to git.
5. Secrets are injected at runtime via Vault Agent or K8S secrets — never baked into images.
6. Audit logging MUST be enabled for all administrative actions. Session recording applies to bastion access (Teleport).
7. Patch cadence: [OWNER TO DEFINE: SLA for applying security patches — e.g., critical within 72h, high within 7d, medium within 30d].
8. Each service MUST have a `docs/hardening.md` documenting its specific hardening configuration.
9. Host hardening baseline: applied via Ansible role — [OWNER TO DEFINE: Lynis score target].
10. Container security scanning (Trivy) MUST run in CI before any image is promoted to PoC or Prod.

## Hardening checklist (per service)

| Check | Tool | Pass condition |
|---|---|---|
| Non-root process | `ps aux` / `docker inspect` | UID != 0 |
| No dev tools in image | `trivy image` | No `gcc`, `make`, `curl`, `wget` flagged unless documented |
| No plaintext secrets | `trivy image --scanners secret` | Zero secret findings |
| Audit logging enabled | Service config + Loki query | Admin actions appear in logs |
| Patch status | Trivy CVE scan | No critical/high CVEs unaddressed beyond SLA |
| Hardening doc exists | `ls tools/{tool}/docs/hardening.md` | File exists and is non-empty |

## Compliance table

| Requirement | Test | Pass condition |
|---|---|---|
| Non-root processes | `docker inspect --format '{{.Config.User}}'` | Non-empty, non-root UID |
| No secrets in images | `trivy image --scanners secret {image}` | Zero findings |
| Audit logging | Loki query for admin events | Admin events present in Loki |
| CVE patch SLA | Trivy output vs patch date | No unpatched critical CVEs beyond defined SLA |
| Hardening doc | File exists per tool | `docs/hardening.md` present and non-stub |

## Override procedure

To override this standard for a specific tool:
1. Create `tools/{tool}/docs/override-hardening.md`
2. State: What is different / Why / Compensating control / Reviewed by @yboujraf
3. PR must include override doc before merge is allowed

## References

- [ISO 27001:2022 A.8.8 — Management of technical vulnerabilities](https://www.iso.org/standard/27001)
- [NIS2 Art. 21(2)(e) — Supply chain security and security in network systems](https://eur-lex.europa.eu/legal-content/EN/TXT/?uri=CELEX%3A32022L2555)
- [CIS Docker Benchmark](https://www.cisecurity.org/benchmark/docker)
- [Lynis hardening tool](https://cisofy.com/lynis/)
- [Trivy container security scanner](https://github.com/aquasecurity/trivy)
