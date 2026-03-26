# Contributing to `{{REPO_NAME}}`

Thanks for contributing.

This repository follows the ORG engineering conventions for naming, delivery, security, and documentation. Keep changes small, reviewable, and traceable.

---

## Before You Start

- Read [README.md](README.md)
- Read [CLAUDE.md](CLAUDE.md) if present
- Check existing ADRs in `docs/adr/`
- Open or link an issue before significant work
- For architecture-impacting changes, create or update an ADR

---

## Branch Naming

Pattern:

```text
{type}/{issue-id}-{short-description}
```

Examples:

```text
feat/42-add-vault-integration
fix/17-dns-resolution-failure
docs/5-update-runbook
ci/12-add-trivy-stage
sec/58-rotate-gpg-keys
```

Allowed branch types:

- `feat`
- `fix`
- `chore`
- `docs`
- `hotfix`
- `release`
- `refactor`
- `test`
- `ci`
- `sec`

Rules:

- lowercase only
- hyphens only
- keep description short, max ~5 words
- one issue per branch unless explicitly agreed

---

## Commit Messages

This repo uses Conventional Commits / Commitizen.

Pattern:

```text
{type}({scope}): {short description}
```

Examples:

```text
feat(platform): add vault bootstrap job
fix(infra): correct vlan assignment for iot segment
docs(doc): add runbook for backup restore
security(sec): tighten container capabilities
ci(platform): add trivy and gitleaks stages
```

Allowed commit types:

- `feat`
- `fix`
- `chore`
- `docs`
- `test`
- `ci`
- `refactor`
- `perf`
- `security`
- `revert`

Allowed scopes usually align with ORG repo scopes:

- `infra`
- `platform`
- `app`
- `svc`
- `lib`
- `tpl`
- `doc`
- `sec`
- `mod`

Breaking changes:

```text
feat(platform)!: replace legacy auth flow
```

or add:

```text
BREAKING CHANGE: ...
```

---

## Merge Request Guidelines

Every MR should be:

- focused on a single concern
- linked to an issue
- easy to review
- safe to roll back

### MR title

Use a concise, descriptive title. Prefer the same style as the squashed commit.

### MR description should include

- **Why** this change exists
- **What** changed
- **How** it was tested
- **Risk / rollback** notes
- **Screenshots / logs** if useful

Suggested template:

```markdown
## Summary
-

## Why
-

## Testing
-

## Risk
-

## Rollback
-

## Closes
- #issue-id
```

---

## Review Expectations

Ask for review when:

- CI is green or failures are explained
- the branch is rebased / merge-ready
- secrets, generated noise, and debug leftovers are removed

Reviewers should check:

- correctness
- readability
- security impact
- operational impact
- naming convention compliance
- documentation / ADR updates where needed

Do not merge if any of these are unresolved:

- failing required pipeline stages
- unclear rollback path for risky changes
- missing docs for operational changes
- missing ADR for major architectural decisions

---

## Documentation Rules

### File naming

Use lowercase, hyphens, English only.

Patterns:

```text
{type}-{subject}-{YYYY-MM-DD}.md
```

Examples:

```text
sow-infra-network-setup-2026-03-25.md
runbook-vault-backup-2026-03-25.md
roadmap-platform-2026-q2-2026-04-01.md
```

### ADRs

Store ADRs under:

```text
docs/adr/
```

Use zero-padded numbering:

```text
0001-use-gitlab-ci.md
0002-vault-as-secret-manager.md
0003-traefik-as-edge-proxy.md
```

Create or update an ADR when changing:

- edge proxy strategy
- secret management approach
- deployment topology
- database/storage architecture
- monitoring/security architecture
- naming or environment standards

---

## Security Requirements

Mandatory rules:

- never commit secrets
- never hardcode credentials, tokens, or private keys
- use Vault for sensitive values
- keep least privilege in CI, runtime, and infra definitions
- prefer rootless builds where supported
- pin versions where practical

Expected checks:

- `gitleaks` for secret scanning
- `trivy` for containers and IaC
- `owasp dependency-check` for app dependencies
- `checkov` for IaC where applicable

If the change affects exposure, auth, networking, or data handling, call it out explicitly in the MR.

---

## Environment Model

Standard track:

```text
dev → test → staging → acceptance → prod
```

Lite track:

```text
dev → staging → prod
```

General rules:

- do not deploy `latest` to production
- production should use immutable semver tags
- environment-specific behavior must be explicit, not hidden in code
- internal services should use `*.org.internal`
- Traefik is the edge proxy; do not introduce ad hoc edge Nginx/Apache patterns

---

## Naming Rules

Use the ORG naming convention consistently.

### Repository

```text
{scope}-{component}-{qualifier}
```

### Internal FQDN

```text
{service}.org.internal
{service}.{env}.org.internal
```

### Device names

```text
{function}-{vendor}-{env}-{number:02d}
```

Examples:

```text
sw-arista-prod-01
srv-proxmox-prod-01
fw-pfsense-prod-01
```

---

## Local Development

Recommended baseline:

- VSCode
- devcontainer
- Docker
- project language toolchain
- access to Vault dev secrets if required

If you add tooling, prefer open-source and self-hostable options aligned with the ORG stack.

---

## Release Process

- merge reviewed changes to `main`
- use conventional commits consistently
- let `release-please` determine semantic version bumps
- keep `CHANGELOG.md` generated, not manually curated unless necessary
- tag releases as:

```text
vMAJOR.MINOR.PATCH
```

Examples:

```text
v1.2.3
v2.0.0-rc.1
```

---

## Questions / Escalation

If you are unsure about:

- naming
- architecture direction
- security impact
- customer-specific deviation from the standard platform

open an issue or draft MR early instead of guessing.
