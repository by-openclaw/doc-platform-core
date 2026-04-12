# ADR-0005: Version Control & CI/CD Strategy

**Date:** 2026-03-27
**Status:** Accepted
**Deciders:** yboujraf, Rune

---

## Context

The platform needs a version control and CI/CD strategy that:
- Works now (before GitLab CE is deployed)
- Scales to full GitLab CE self-hosted when ready
- Provides clear separation of concerns between teams and repos
- Does not require expensive paid plans

---

## Decision

### Phase 1 — GitHub `by-openclaw` (current → GitLab CE deployed)

- Single GitHub org: `by-openclaw` (keep name, no rename — avoids webhook breakage)
- Access control: per-repo collaborators (GitHub free, no Teams needed)
- CI: GitHub Actions (thin workflows, Discord notify, basic lint)
- Repos follow naming convention: `{scope}-{component}-{qualifier}`
- I (Rune) operate with one org-scoped PAT stored in `infra/secrets/`

### Phase 2 — GitLab CE self-hosted (target state)

GitLab CE deployed on `srv-proxmox-poc-01`, FQDN `gitlab.by-systems.arpa`.

**Subgroup structure:**
```
gitlab.by-systems.arpa/
  by-systems/
    infra/       ← proxmox-ansible, network-arista, terraform-proxmox, terraform-contabo
    lib/         ← synology-dsm, proxmox-api, (future protocol libs)
    platform/    ← netbox-config, vault-config, authentik-config, gitlab-config
    sec/         ← wazuh-config, openscap-policies
    mod/         ← broadcast-ptp, voip-kamailio, cctv-frigate
    doc/         ← platform-core (this repo)
    tpl/         ← pipeline-base, repo-infra, repo-app
```

**Access control via GitLab groups:**
- `infra` subgroup → infra team only
- `lib` subgroup → all engineers (shared libs)
- `platform` subgroup → platform team
- `sec` subgroup → CISO/security team
- External users (customers/freelancers) → invited per-project only, no subgroup access

**Authentication:** Authentik OIDC → GitLab SAML (users log in with Microsoft/Authentik credentials)

### CI/CD migration

- GitHub Actions workflows → replaced with GitLab CI pipelines
- Shared pipeline templates in `tpl/pipeline-base` (DRY — all repos inherit)
- Self-hosted GitLab Runner on PoC Proxmox (unlimited CI minutes)
- GitHub kept as read-only mirror (optional) or decommissioned

### GitHub → GitLab migration steps (when ready)

1. Deploy GitLab CE (Docker Compose on `srv-proxmox-poc-01`)
2. Configure Authentik SAML integration
3. Create subgroup structure
4. Mirror all `by-openclaw` repos → `by-systems/` subgroups
5. Switch default remote in all workspace clones
6. Rewrite CI pipelines (mechanical, template-driven)
7. Update Discord webhooks to GitLab
8. Decommission GitHub Actions workflows

---

## CISO mapping

> Applies only to controls directly relevant to this ADR's scope.

### ISO/IEC 27001:2022

| Control | Title | Status | Notes |
|---|---|---|---|
| A.8.32 | Change management | ✓ Covered | Migration path and CI ownership are documented in phased form |
| A.8.9 | Configuration management | ✓ Covered | Repos, templates, and CI are centrally organised and version-controlled |
| A.5.23 | Information security for use of cloud services | ✓ Covered | GitHub is interim only; GitLab CE target is self-hosted |

### NIS2 (Directive 2022/2555)

| Article | Requirement | Status | Notes |
|---|---|---|---|
| Art. 21(2)(e) | Security in network and information systems acquisition, development and maintenance | ✓ Covered | CI/CD migration is controlled and template-driven |

### GDPR (Regulation 2016/679)

Not applicable — this ADR covers VCS and CI strategy, not personal data processing.

## Consequences

- No cost increase in either phase
- Full separation of concerns via GitLab subgroups (Phase 2)
- CI rewrite required on migration — acceptable (templates make it fast)
- Discord webhook URLs change on migration — requires update
- GitHub org `by-openclaw` kept as interim, not renamed
