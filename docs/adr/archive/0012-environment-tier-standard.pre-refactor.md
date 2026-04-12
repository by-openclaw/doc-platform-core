# ADR-0012: Environment Tier Standard

**Status:** Accepted
**Date:** 2026-03-31
**Deciders:** @yboujraf

## Context

The platform uses multiple environments for isolation (lab, development, testing, production). Without a formal tier standard, environment labels are inconsistent — some systems use `nonprod`, others `sandbox`, others omit the label entirely and assume "unlabeled = production." This is dangerous: an unlabeled resource could be treated as either throwaway or production depending on who's looking.

The naming & identity convention defines six environment tiers. This ADR extracts the tier standard as an independently referenceable decision, since it applies to every layer of the platform: infrastructure, credentials, DNS, certificates, service accounts, and monitoring.

## Decision

### Five environment tiers — ordered pipeline

> **Note (2026-04-03):** `poc` has been removed as an environment tier. It was being confused with the Proxmox node name `srv-proxmox-poc-01` (hardware label). `poc` is not an env tier — it is infrastructure naming. All VMs deployed on that node are `env=prod` (or `dev`/`test` as applicable). Env is always declared per-VM/LXC, never inferred from node or folder names.

```
dev → test → staging → acc → prod
```

| Tier | Full name | Purpose | Promotion gate |
|---|---|---|---|
| `dev` | Development | Active development | CI green |
| `dev` | Development | Active development, feature branches, local integration | CI green (lint + unit tests) |
| `test` | Test | Automated testing — integration, regression, smoke | Full test suite green |
| `staging` | Staging | Pre-prod validation — mirrors prod config | Deployment dry-run successful |
| `acc` | Acceptance / UAT | Customer or stakeholder validation | Business sign-off |
| `prod` | Production | Live — SLA applies, monitoring active, alerting enabled | Change advisory board (CAB) approval |

### Rules

1. **`prod` is always explicit.** No label = something is wrong, not "it's prod." Every resource, credential, hostname, and service account must carry its environment tier label.

2. **No implicit environments.** If a resource does not have an env label, it is non-compliant. The only exception is infrastructure-global resources (e.g., a single NAS shared across all envs), which omit the env suffix and document the reason.

3. **Environment label position is fixed.** The env label appears in the same position across all naming layers. Consistency over convenience.

4. **Credentials are per-environment.** A `poc` token cannot access `prod` resources. Service accounts are `svc-{function}-{env}` — separate credentials per tier.

5. **No shortcutting the pipeline.** Code does not go from `dev` directly to `prod`. Each tier has a promotion gate. Skipping tiers requires explicit CAB approval and is logged as a risk in RAID.md.

6. **Certificates:** One wildcard cert: `*.by-systems.be` (covers all envs). Issued via Let's Encrypt Cloudflare DNS-01.

### Environment-specific constraints

| Tier | Data | Backup | Monitoring | Access |
|---|---|---|---|---|

| `dev` | Synthetic/test only | Daily (best effort) | Basic | All developers |
| `test` | Synthetic/test only | None required | CI integration | CI service accounts |
| `staging` | Anonymized prod copy | Daily | Full (mirrors prod) | Ops + senior devs |
| `acc` | Anonymized prod copy | Daily | Full | Stakeholders + ops |
| `prod` | Real data | Per retention policy | Full + alerting | `grp-prod-admin` (max 2-3) |

## Consequences

**Positive:**
- Every resource is unambiguously tied to an environment — no "which env is this?" questions during incidents
- Credential blast radius is contained per tier — compromised `poc` token cannot affect `prod`
- Pipeline is explicit — promotion gates prevent accidental deployments
- Compliance auditors can verify env separation by checking naming alone

**Negative:**
- Six tiers multiply infrastructure cost (6x VMs, 6x credentials, 6x certificates at full scale)
- Not all tiers are needed immediately — `poc` and `prod` are sufficient for Phase 1. Intermediate tiers are activated as the platform matures.
- Infrastructure-global resources (shared NAS, shared DNS) are exceptions that must be documented

## CISO mapping

> Applies only to controls directly relevant to this ADR's scope.

### ISO/IEC 27001:2022

| Control | Title | Status | Notes |
|---|---|---|---|
| A.8.31 | Separation of development, test and production environments | ✓ Covered | Six explicit tiers with defined promotion gates |
| A.8.25 | Secure development lifecycle | ✓ Covered | Dev/test environments isolated from prod |
| A.8.33 | Test information | ⚠ Partial | Policy defined (prod data anonymized for lower tiers); enforcement not yet automated |

### NIS2 (Directive 2022/2555)

| Article | Requirement | Status | Notes |
|---|---|---|---|
| Art. 21(2)(a) | Risk management | ✓ Covered | Environment separation is a primary blast-radius reduction control |
| Art. 21(2)(e) | Security in network and information systems | ✓ Covered | Per-env credentials and certificates enforce separation at every layer |

### GDPR (Regulation 2016/679)

| Article | Requirement | Status | Notes |
|---|---|---|---|
| Art. 25 | Data protection by design and by default | ⚠ Partial | Test environments use synthetic data policy defined; no prod data yet to protect |
| Art. 32 | Security of processing | ✓ Covered | Environment isolation prevents accidental data exposure across tiers |

## Notes

- Source: naming & identity convention draft §1 (promoted to its own ADR for independent referenceability)
- Current state (Phase 1): `prod` tier is active. All current VMs on `srv-proxmox-poc-01` are `env=prod`. `dev` / `test` tiers activate when the first non-prod workload needs isolation.
- Environment pipeline: `poc → dev → test → staging → acc → prod` (OPERATING-STANDARD.md §9.4)
