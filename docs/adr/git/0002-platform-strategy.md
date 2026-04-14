# git/0002 — Platform Strategy (GitHub → GitLab CE)

**Status:** Draft
**Date:** 2026-04-12 (supersedes flat ADR-0005, 2026-03-27)
**Scope:** Where git repos live and where CI runs, now and target.
**Related:** `git/0001-workflow`, `identity/0003-machine-credentials`, `infra/0002-platform-charter`

---

## Context

The platform needs a VCS + CI strategy that works today (no paid plans, no SaaS lock-in) and scales to a self-hosted solution under BY-SYSTEMS control. The workflow in `git/0001` is host-agnostic — this ADR defines the **hosts**.

## Decision

### Current state — GitHub `by-openclaw`

- Single org: `by-openclaw` (kept as-is, no rename — avoids webhook/clone breakage)
- Plan: **GitHub Free**. No upgrade planned.
- Access control: per-repo collaborators (Teams not available on Free)
- Repos: all private
- CI: GitHub Actions (thin workflows — release-please, lint, Discord notify)
- CI credentials: repo-level secrets (`RUNE_GITHUB_TOKEN`) per `identity/0003-machine-credentials`
- GitHub remains for public open-source mirror repos even after migration

### Target state — GitLab CE self-hosted

- Deployed on Proxmox. FQDN finalized at install time (candidate: `gitlab.by-systems.arpa`) — update this ADR when the service is deployed.
- **All CI runs on self-hosted GitLab Runner** — unlimited minutes, no SaaS quotas
- Auth via Authentik OIDC (see `identity/0001-authentication`)
- Group-level CI/CD variables (`RUNE_GITLAB_TOKEN`) — see `identity/0003-machine-credentials`
- GitHub kept only for public-facing repos (open-source mirrors); everything else moves

### GitLab subgroup structure

```
<gitlab-fqdn>/by-openclaw/
├── infra/       # ansible-platform, infra-terraform, etc.
├── lib/         # lib-opnsense, lib-synology-dsm, future protocol libs
├── platform/    # netbox-config, vault-config, authentik-config, gitlab-config
├── sec/         # wazuh-config, openscap-policies
├── mod/         # broadcast-ptp, voip-kamailio, cctv-frigate (future)
├── doc/         # doc-platform-core (this repo)
└── tpl/         # pipeline-base, repo-infra, repo-app
```

### Subgroup access model

| Subgroup | Default access |
|---|---|
| `infra`, `sec` | Platform/security team only |
| `lib`, `tpl` | All engineers (shared libraries, pipeline templates) |
| `platform`, `doc` | Platform team |
| `mod` | Module-specific teams |

External users (customers, freelancers) are invited **per project**, never at subgroup level.

### CI migration

- GitHub Actions workflows → GitLab CI pipelines
- Shared templates in a **dedicated `ci-templates` repo** at the org level (see `naming/0004-automation §1` for the `ci-templates` repo type)
- Self-hosted GitLab Runner on Proxmox
- Release-please stays the pattern, re-implemented as GitLab CI template

### CI templates include pattern

Every `svc-*`, `infra-*`, `lib-*`, and `ansible-*` repo's `.gitlab-ci.yml` includes from the central `ci-templates` repo using GitLab CI's `include:` directive:

```yaml
# .gitlab-ci.yml in any platform repo
include:
  - project: by-openclaw/ci-templates
    file: .gitlab-ci/build.yml
  - project: by-openclaw/ci-templates
    file: .gitlab-ci/deploy.yml
  - project: by-openclaw/ci-templates
    file: .gitlab-ci/security.yml
```

**Rules:**
- **No copy-paste.** A repo's `.gitlab-ci.yml` **never** duplicates content from `ci-templates` — it includes by reference.
- **One source of truth** for CI stages across the platform. Updating a template in `ci-templates` propagates to every downstream repo on its next pipeline run.
- **Template files follow the layout** `ci-templates/.gitlab-ci/{concern}.yml` — one file per concern (build, test, deploy, security, lint, release).
- **Include order matters** — security template runs after build + test but before deploy. Templates declare their own `stage:` in jobs to enforce order.
- Repos that don't use GitLab CI (currently GitHub Actions) ignore the `ci-templates` repo until they migrate — the include is a one-line change at migration time.

### Migration steps (executed when GitLab CE is deployed)

1. Deploy GitLab CE (Docker Compose on Proxmox)
2. Configure Authentik SAML/OIDC
3. Create subgroup structure
4. Mirror `by-openclaw/*` private repos → `by-openclaw/<subgroup>/` on GitLab
5. Switch default remote in all workspace clones
6. Rewrite CI pipelines using `tpl/pipeline-base`
7. Update Discord webhooks to GitLab
8. Decommission private-repo GitHub Actions workflows
9. Keep public open-source mirrors on GitHub

## Consequences

- No cost increase in either state — GitHub Free + GitLab CE are both free
- CI is rewritten once during migration (mechanical, template-driven)
- Self-hosted GitLab Runner removes SaaS CI minute quotas entirely
- Discord webhook URLs change on migration — one-time update
- Public open-source repos stay on GitHub for community visibility and issue tracking
- Branch protection, PR workflow, and config (see `git/0001`, `git/0003`) are identical on both hosts

## CISO mapping

| Framework | Controls covered |
|---|---|
| ISO 27001:2022 | A.5.23, A.8.9, A.8.32 |
| NIS2 | Art. 21(2)(e) |
