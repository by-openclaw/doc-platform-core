# ORG Platform — RAID Log

**Last updated:** 2026-03-25
**Status:** Active — PoC phase
**Related docs:** `docs/stack.md`, `docs/roadmap.md`, `docs/architecture.md`

> RAID = **R**isks · **A**ssumptions · **I**ssues · **D**ependencies
> This is a living document. Update on every sprint/phase review.

---

## R — Risks

| ID | Risk | Probability | Impact | Severity | Mitigation | Owner | Status |
|---|---|---|---|---|---|---|---|
| R-01 | Proxmox VE single node — no HA during PoC | High | High | 🔴 Critical | Accept for PoC; plan multi-node cluster before prod. Document recovery runbook. | Infra | Open |
| R-02 | PostgreSQL single instance during PoC (no Patroni) | High | High | 🔴 Critical | Accept for PoC; migrate to Patroni HA cluster in Phase 2.5. Daily pg_dump to MinIO. | Platform | Open |
| R-03 | HashiCorp Vault seal/unseal — data loss if node lost without backup | Medium | Critical | 🔴 Critical | Vault auto-unseal via cloud KMS or Shamir shares stored in Vaultwarden. Snapshot policy: daily to MinIO. | Platform | Open |
| R-04 | Teleport CE session recording storage growth | Low | Medium | 🟡 Medium | Set retention policy (90 days). Sessions → MinIO. Monitor storage via Prometheus. | Platform | Open |
| R-05 | Kaniko build secrets leak via `--build-arg` | Medium | High | 🔴 Critical | Policy enforced: `--build-arg` forbidden for secrets. Trivy post-build scan catches leaks. Gitleaks on Dockerfile. | DevOps | Open |
| R-06 | Nexus OSS disk exhaustion from cache growth | Medium | Medium | 🟡 Medium | Blob store cleanup policy: remove unused after 30 days. Alert on disk > 80%. | Platform | Open |
| R-07 | GitLab LFS storage growth (assets, media) | Medium | Low | 🟢 Low | LFS → MinIO backend. Quota per project. Monitor via Prometheus. | Platform | Open |
| R-08 | Anthropic/OpenAI API overload during heavy agent use | High | Low | 🟡 Medium | Fallback chain configured (Anthropic → OpenAI). Token tied to MAX plan. Monitor via OpenClaw `/stats`. | DevOps | Open |
| R-09 | Arista EOS misconfiguration causing broadcast storm | Low | Critical | 🟡 Medium | Change control via Ansible only. No manual EOS edits in prod. Pre-apply dry-run (`--check`). | Network | Open |
| R-10 | Authentik SSO outage blocks all platform access | Low | Critical | 🔴 Critical | Local admin accounts on each service as break-glass. Authentik HA in prod (Phase 2.5). | Platform | Open |
| R-11 | Teleport CA key loss — all SSH certs revoked | Low | Critical | 🔴 Critical | Teleport CA backed up to Vault. Rotation procedure documented in runbook. | Security | Open |
| R-12 | Nexus upstream proxy blocked (corporate firewall / ISP) | Low | Medium | 🟡 Medium | Nexus configured to use HTTP proxy if needed. pfSense egress rules explicitly allow nexus outbound. | Network | Open |
| R-13 | License drift — tool switches from OSS to paid | Low | Medium | 🟡 Medium | All tools pinned to OSS/CE/Community editions. Version locked in Ansible/Helm. Monitored via ADR review. | Platform | Open |
| R-14 | PII data leak via unredacted screenshots/logs in Git | Medium | High | 🔴 Critical | Presidio pre-commit hook + CI scan. Asset naming convention enforces redaction check before commit. | Security | Open |

---

## A — Assumptions

| ID | Assumption | Impact if wrong | Validation |
|---|---|---|---|
| A-01 | Bare metal hardware is available and racked before Phase 1 starts | Phase 1 blocked | Confirm hardware delivery date |
| A-02 | Arista switches support EOS API (eAPI) and are reachable from Ansible | Network automation blocked | Verify EOS version and eAPI enabled on all switches |
| A-03 | Internet uplink provides stable connectivity for initial package pulls (Nexus cold cache) | CI builds fail on first run | Test connectivity from Nexus host to npmjs.org, nuget.org, pkg.go.dev, pypi.org |
| A-04 | Cloudflare is available as public DNS provider and DNS-01 ACME works | Public cert issuance blocked | Verify Cloudflare zone ownership and API token |
| A-05 | Proxmox VE Community (no subscription) is acceptable for PoC | No enterprise support | Accepted — PoC scope |
| A-06 | GitLab CE (MIT) covers all CI/CD requirements without EE features | May need EE for advanced compliance features | Review feature gap before prod promotion |
| A-07 | Teleport CE provides sufficient audit logging for ISO 27001 / NIS2 | May need Teleport Enterprise for advanced compliance features | Validate against compliance checklist in Phase 3 |
| A-08 | k3s is sufficient for PoC workloads (not full K8S) | May hit k3s limitations at scale | Evaluate at Phase 2.5; migration path to RKE2/Talos documented |
| A-09 | HashiCorp Vault OSS is sufficient (no Vault Enterprise needed) | Namespace isolation, HSM, and some enterprise features unavailable | Accepted for PoC; re-evaluate at prod |
| A-10 | One PostgreSQL cluster (Patroni) can host all platform services (GitLab, Vault, Authentik, NetBox, Zabbix) | Resource contention; requires DB separation | Monitor per-DB resource usage; separate clusters if needed |
| A-11 | Mailcow on `example.com` (RFC 2606) is sufficient for non-prod email testing | May need real domain for some tests | Validated: RFC 2606 reserved, no external leak risk |
| A-12 | ORG team has Ansible and Terraform skills for Phase 1 IaC | Phase 1 delayed | Skills assessment before start |
| A-13 | Nexus OSS (Apache 2.0) remains free for all required formats | Forced migration to paid tier | Monitor Sonatype licensing changes; Nexus pinned to current OSS version |
| A-14 | Customer production mail (Exchange/Google) will be available for SMTP relay config | Platform notification emails fail in prod | Collect SMTP relay credentials during customer onboarding |

---

## I — Issues

| ID | Issue | Severity | Date raised | Resolution | Status |
|---|---|---|---|---|---|
| I-01 | OpenClaw Anthropic token was OpenClaw shared pool (not MAX plan) — caused overload errors | High | 2026-03-25 | Re-ran `openclaw models auth setup-token --provider anthropic` — new token tied to MAX plan. Old API key removed from config and revoked. | ✅ Resolved |
| I-02 | Discord WebSocket instability (code 1006, 520) — intermittent reconnects | Low | 2026-03-25 | Discord-side transient issue. Gateway auto-recovered. Monitor for recurrence. | ✅ Resolved (monitoring) |
| I-03 | `openclaw gateway restart` kills agent mid-command (self-restart) | Low | 2026-03-25 | Workaround: use `systemctl --user restart openclaw-gateway.service` from terminal. | ⚠️ Workaround |
| I-04 | `ANTHROPIC_API_KEY` ([REDACTED]) was stored in session history JSONL | Medium | 2026-03-25 | Key revoked on Anthropic console. Session log is local-only. Config files cleaned. | ✅ Resolved |
| I-05 | Template files used `.md.template` extension — not rendered by editors | Low | 2026-03-25 | Renamed all templates to `.tpl.md`. Convention documented in naming-convention.md §5. | ✅ Resolved |
| I-06 | Verdaccio and Athens identified as gaps — npm/Go proxy only, not multi-format | Medium | 2026-03-25 | Replaced by Nexus OSS in stack decision. ADR and roadmap updated. | ✅ Resolved |

---

## D — Dependencies

| ID | Dependency | Type | Required by | Risk if unavailable | Owner |
|---|---|---|---|---|---|
| D-01 | Bare metal server(s) for Proxmox | Hardware | Phase 1 | Phase 1 blocked | Infra |
| D-02 | Arista switches (7020/7060/7048) | Hardware | Phase 1 | Network automation blocked | Network |
| D-03 | UniFi APs | Hardware | Phase 1 | Wireless access blocked | Network |
| D-04 | ISP uplink (static IP or DDNS) | External service | Phase 1 | Public access blocked | Infra |
| D-05 | Cloudflare account + API token | External service | Phase 1 | Public DNS + ACME blocked | Infra |
| D-06 | Anthropic MAX plan (OpenClaw agent) | External service | Platform tooling | Agent falls back to OpenAI | DevOps |
| D-07 | GitLab CE Docker image (hub.docker.com → Nexus mirror) | Software | Phase 2 | Bootstrap CI blocked | Platform |
| D-08 | Teleport CE binary / Helm chart | Software | Phase 2 | Bastion deployment blocked | Security |
| D-09 | Nexus OSS Docker image | Software | Phase 2 | Dep proxy unavailable; CI builds hit internet directly | Platform |
| D-10 | Customer SMTP relay credentials (Exchange/Google) | Customer-provided | Phase 5 | Platform notification emails fail in prod | Customer |
| D-11 | Customer network access (firewall rules, VPN) for site-to-site | Customer-provided | Phase 5 | Customer site integration blocked | Network |
| D-12 | Vault backup storage (MinIO) | Internal service | Phase 2 | Vault snapshot restore impossible | Platform |
| D-13 | Authentik SSO up before GitLab/Vault/NetBox OIDC wiring | Service ordering | Phase 2 | OIDC config fails | Platform |
| D-14 | PostgreSQL up before GitLab/Vault/Authentik/NetBox | Service ordering | Phase 2 | Service startup fails | Platform |
| D-15 | Nexus cache seeded before CI pipelines run at scale | Service readiness | Phase 2 | First pipeline runs hit internet; slow + brittle | Platform |
| D-16 | Teleport deployed before pfSense blocks direct SSH | Service ordering | Phase 2 | Engineers locked out if firewall rule applied early | Security |

---

## Review cadence

| Phase | RAID review |
|---|---|
| Phase 1 (Foundation) | Weekly during active work |
| Phase 2 (Platform Services) | Weekly |
| Phase 3+ | Bi-weekly |
| Production promotion | Full RAID review required before go-live |

---

## Status legend

| Symbol | Meaning |
|---|---|
| 🔴 Critical | Immediate attention required |
| 🟡 Medium | Monitor and plan mitigation |
| 🟢 Low | Accept or defer |
| ✅ Resolved | Closed |
| ⚠️ Workaround | Mitigated, not fixed |
| 🔵 Deferred | Accepted for later phase |
