# ADR-0012: Environment Tier Standard

**Status:** Accepted
**Date:** 2026-03-31
**Deciders:** @yboujraf

## Context

The platform uses multiple environments for isolation (lab, development, testing, production). Without a formal tier standard, environment labels are inconsistent — some systems use `nonprod`, others `sandbox`, others omit the label entirely and assume "unlabeled = production." This is dangerous: an unlabeled resource could be treated as either throwaway or production depending on who's looking.

The naming & identity convention (ADR-0010 §1) defines six environment tiers. This ADR extracts the tier standard as an independently referenceable decision, since it applies to every layer of the platform: infrastructure, credentials, DNS, certificates, service accounts, and monitoring.

## Decision

### Six environment tiers — ordered pipeline

```
poc → dev → test → staging → acc → prod
```

| Tier | Full name | Purpose | Promotion gate |
|---|---|---|---|
| `poc` | Proof of concept | Lab / sandbox — pre-pipeline, no SLA, disposable | Manual — "does this idea work?" |
| `dev` | Development | Active development, feature branches, local integration | CI green (lint + unit tests) |
| `test` | Test | Automated testing — integration, regression, smoke | Full test suite green |
| `staging` | Staging | Pre-prod validation — mirrors prod config | Deployment dry-run successful |
| `acc` | Acceptance / UAT | Customer or stakeholder validation | Business sign-off |
| `prod` | Production | Live — SLA applies, monitoring active, alerting enabled | Change advisory board (CAB) approval |

### Rules

1. **`prod` is always explicit.** No label = something is wrong, not "it's prod." Every resource, credential, hostname, and service account must carry its environment tier label.

2. **No implicit environments.** If a resource does not have an env label, it is non-compliant. The only exception is infrastructure-global resources (e.g., a single NAS shared across all envs), which omit the env suffix and document the reason.

3. **Environment label position is fixed.** The env label appears in the same position across all naming layers (see ADR-0010 §2). Consistency over convenience.

4. **Credentials are per-environment.** A `poc` token cannot access `prod` resources. Service accounts are `svc-{function}-{env}` — separate credentials per tier (ADR-0010 §3.2).

5. **No shortcutting the pipeline.** Code does not go from `dev` directly to `prod`. Each tier has a promotion gate. Skipping tiers requires explicit CAB approval and is logged as a risk in RAID.md.

6. **Certificates:** One wildcard cert: `*.by-systems.be` (covers all envs). Issued via Let's Encrypt Cloudflare DNS-01.

### Environment-specific constraints

| Tier | Data | Backup | Monitoring | Access |
|---|---|---|---|---|
| `poc` | Synthetic/test only | None required | Optional | All developers |
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

## Compliance

- **ISO A.12.1.4** (separation of development, testing, and operational environments) — six explicit tiers with defined promotion gates
- **ISO A.14.2.6** (secure development environment) — dev/test environments isolated from prod
- **ISO A.14.3.1** (protection of test data) — prod data anonymized before use in staging/acc
- **NIS2 Art.21(2)(a)** (risk management) — env separation as risk mitigation control
- **NIS2 Art.21(2)(e)** (security in network and information systems acquisition) — per-env credentials and certificates
- **GDPR Art.25** (data protection by design) — test environments use synthetic data, prod data anonymized for lower tiers
- **GDPR Art.32** (security of processing) — env isolation prevents accidental data exposure

## Notes

- Source: naming & identity convention draft §1 (promoted via ADR-0010)
- This ADR is extracted from ADR-0010 for independent referenceability — other ADRs and CLAUDE.md files reference it directly
- Current state (Phase 1): only `poc` tier is active. `prod` tier activates with first production workload.
- Environment pipeline: `poc → dev → test → staging → acc → prod` (OPERATING-STANDARD.md §9.4)
