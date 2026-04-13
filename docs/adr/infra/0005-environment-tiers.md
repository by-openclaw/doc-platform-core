# infra/0005 — Environment Tiers

**Status:** Draft
**Date:** 2026-04-13 (supersedes flat ADR-0012, 2026-03-31)
**Scope:** Authoritative definition of environment tiers used across every platform layer — hosts, credentials, DNS, certificates, service accounts, backups, monitoring. Every asset on the platform carries exactly one tier.
**Related:** `naming/0001-infra`, `naming/0002-identity`, `security/0001-secret-storage`, `infra/0004-network-architecture`, `infra/0008-backup-strategy`

---

## Context

The platform uses multiple environments for isolation. Without a formal tier standard, environment labels drift across tools — some systems use `nonprod`, others `sandbox`, others omit the label and assume "unlabeled = production". This is dangerous: an unlabeled resource could be treated as either throwaway or production depending on who's looking.

Flat ADR-0012 originally defined six tiers including `poc`. That was a mistake — `poc` was a hostname label on the historical Proxmox node `srv-proxmox-poc-01`, not an environment tier. Conflating the two meant operators couldn't tell whether `poc` referred to the physical node, the env tier, or both. This ADR removes `poc` entirely and adds `drp` (disaster recovery) as the sixth tier.

## Decision

### Six tiers — ordered pipeline

```
dev → test → staging → acc → prod   (the pipeline)
                              └── drp   (restored from prod, DR-only)
```

| Tier | Full name | Purpose | Promotion gate |
|---|---|---|---|
| `dev` | Development | Active development, feature branches, local integration | CI green (lint + unit tests) |
| `test` | Test | Automated testing — integration, regression, smoke | Full test suite green |
| `staging` | Staging | Pre-prod validation — mirrors prod config | Deployment dry-run successful |
| `acc` | Acceptance / UAT | Customer or stakeholder validation | Business sign-off |
| `prod` | Production | Live — SLA applies, monitoring active, alerting enabled | Change advisory board (CAB) approval |
| `drp` | Disaster recovery | Stand-by copy of prod, restored from backup, activated during failover | N/A — DR activation, not promotion |

**Removed tier:** `poc` was in flat ADR-0012. It conflated the historical Proxmox node name (`srv-proxmox-poc-01`, a hardware label) with an environment tier. `dev` already covers the "proof of concept / throwaway experiment" use case. `poc` is no longer an env tier and must not appear in any new asset, credential, DNS zone, secret path, or service account name.

**Added tier:** `drp` is the disaster recovery environment. It mirrors `prod` topology (same VLANs, same service URLs on a failover DNS zone per `naming/0001-infra §7`) but holds **restored data** — often slightly stale compared to prod. During normal operation `drp` is read-only; during failover it becomes the live tier.

### Rules

1. **`prod` is always explicit.** No label = something is wrong, not "it's prod". Every resource, credential, hostname, service account, and secret path carries its environment tier label.

2. **No implicit environments.** If a resource does not have an env label, it is non-compliant. The only exception is infrastructure-global resources (e.g. a single shared NAS) which omit the env suffix and document the reason in the asset description.

3. **Environment tier is per-asset, not per-node.** The Proxmox node a VM runs on does NOT set the VM's env — a physical node can host `env=prod` VMs, `env=dev` VMs, and `env=test` VMs simultaneously. Env is declared on the VM itself (NetBox custom field, Terraform variable, Ansible host_var).

4. **Credentials are per-environment.** A `dev` token cannot access `prod` resources. Service accounts follow `svc-{function}-{env}` per `naming/0002-identity §3`. Vault paths follow `secret/{env}/{service}/{key}` per `security/0001-secret-storage`.

5. **No shortcutting the pipeline.** Code does not go from `dev` directly to `prod`. Each tier has a promotion gate. Skipping tiers requires explicit CAB approval and is logged as a risk in `RAID.md`.

6. **Certificates per tier.** One wildcard cert per DNS zone per tier — prod uses `*.{domain}`, other tiers use `*.{env}.{domain}`. Covered in `security/0004-certificate-strategy`.

7. **`drp` is not a promotion target.** Code is not "promoted to DR" — DR is populated by restoring prod backups and replaying Terraform / Ansible against the DR infrastructure. Failover is a runbook operation, not a pipeline step.

### Tier-specific constraints

| Tier | Data | Backup | Monitoring | Access |
|---|---|---|---|---|
| `dev` | Synthetic / test only | Daily (best effort) | Basic | All developers |
| `test` | Synthetic / test only | None required | CI integration | CI service accounts |
| `staging` | Anonymized prod copy | Daily | Full (mirrors prod) | Ops + senior devs |
| `acc` | Anonymized prod copy | Daily | Full | Stakeholders + ops |
| `prod` | Real data | Per `infra/0008-backup-strategy` | Full + alerting | `adm_*` accounts only (max 2-3 per asset) |
| `drp` | Restored prod data (may be stale) | Not backed up itself — `drp` is the backup | Full (identical to prod) | Break-glass + DR team |

### Infrastructure mapping

| Tier | Proxmox SDN zone (per `infra/0004-network-architecture §7`) | DNS zone (per `naming/0001-infra §7`) | Vault path (per `security/0001-secret-storage`) |
|---|---|---|---|
| `dev` | `test` | `*.dev.{domain}` | `secret/dev/...` |
| `test` | `test` | `*.test.{domain}` | `secret/test/...` |
| `staging` | `test` | `*.staging.{domain}` | `secret/staging/...` |
| `acc` | `test` | `*.acc.{domain}` | `secret/acc/...` |
| `prod` | `prod` | `*.{domain}` (no sub-domain) | `secret/prod/...` |
| `drp` | `prod` | `*.drp.{domain}` | `secret/drp/...` |

**Note:** there are **two SDN zones** (`prod` and `test`) serving **six env tiers**. Prod zone hosts both `prod` and `drp` (`drp` inherits prod's network topology). Test zone hosts `dev`/`test`/`staging`/`acc` — four tiers sharing one SDN zone because they differ only in promotion gate and access, not in network topology.

**Non-active tiers are reserved** — their tokens cannot be used for anything else and their infrastructure slots (VLAN ranges, DNS zones, Vault paths) are pre-allocated at design time, even before the tier is populated.

## Consequences

- **Every asset is unambiguously tied to an environment** — no "which env is this?" questions during incidents.
- **Credential blast radius is contained per tier** — compromised `dev` token cannot affect `prod`.
- **Pipeline is explicit** — promotion gates prevent accidental deployments.
- **Compliance auditors can verify env separation by checking naming alone** — NetBox + Vault + DNS all carry the tier consistently.
- **Six tiers multiply infrastructure cost** (6× VMs, 6× credentials, 6× certificates at full scale). Mitigated by activating tiers only when needed — current state has only `prod` fully live.
- **Infrastructure-global resources (shared NAS, shared DNS)** are exceptions that must be documented in the asset description.
- **`poc` is retired** — any surviving references to `env=poc` in NetBox, Vault paths, DNS, or service account names are migration backlog items tracked outside this ADR
- **`drp` is a first-class tier** — backup strategy, cert issuance, network topology all account for it from the start

## Revision triggers

Revise when:
- A 7th tier is required (unlikely — this 6-tier model covers dev-through-DR)
- `drp` is replaced by a different DR model (e.g. active-active prod + prod')
- The promotion pipeline changes (e.g. `staging` and `acc` merge for simpler customer engagements)
- Infrastructure-global resources become common enough to warrant a `global` tier of their own
- Multi-tenant deployments require per-tenant env isolation beyond the tier token (e.g. `prod-client-xyz`)

## CISO mapping

| Framework | Controls covered |
|---|---|
| ISO 27001:2022 | A.8.31 (separation of development, test and production environments — 6 explicit tiers), A.8.25 (secure development lifecycle — dev/test isolated from prod), A.8.33 (test information — anonymized prod data policy) |
| NIS2 | Art. 21(2)(a) (risk management — env separation is a primary blast-radius reduction control), Art. 21(2)(c) (business continuity — `drp` is first-class), Art. 21(2)(e) (network and information systems security — per-env credentials and certificates) |
| GDPR | Art. 25 (data protection by design — test environments use synthetic data), Art. 32 (security of processing — env isolation prevents accidental data exposure) |
