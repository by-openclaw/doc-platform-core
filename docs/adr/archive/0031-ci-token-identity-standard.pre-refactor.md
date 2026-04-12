# ADR-0031 — CI Token & Identity Standard

**Date:** 2026-04-05
**Status:** Accepted
**Deciders:** Youssef Boujraf
**Tags:** ci, security, identity, tokens

## Context

CI/CD workflows need tokens (PATs) to perform mutations on GitHub (create releases, update project boards, push tags). Currently the platform uses inconsistent secret names (`GH_TOKEN`, `PROJECT_TOKEN`, `GITHUB_TOKEN`) with no identity traceability. Some secrets are duplicated at both org and repo level. Nobody can tell whose PAT is stored where.

This ADR defines the naming convention, ownership rules, and scope constraints for CI tokens across all repositories.

## Decision

### Naming convention

Pattern: `{IDENTITY}_{PLATFORM}_TOKEN`

- `IDENTITY` = service account or human handle, UPPER_SNAKE_CASE (e.g. `RUNE`, `YBOUJRAF`, `OPUS`)
- `PLATFORM` = target system (e.g. `GITHUB`, `GITLAB`, `VAULT`, `OPNSENSE`)
- `TOKEN` = literal suffix

Examples:
| Secret name | Identity | Platform | Purpose |
|---|---|---|---|
| `RUNE_GITHUB_TOKEN` | by-rune | GitHub | CI: releases, project boards, workflow mutations |
| `YBOUJRAF_GITHUB_TOKEN` | yboujraf | GitHub | Manual: repo creation, org admin (if needed) |
| `RUNE_OPNSENSE_TOKEN` | by-rune | OPNsense | Integration tests: API key:secret |
| `RUNE_GITLAB_TOKEN` | by-rune | GitLab CE | Future: GitLab CI |
| `RUNE_VAULT_TOKEN` | by-rune | Vault | Future: secret retrieval via AppRole |

### Rules

1. **One token per identity per platform.** No user or service account may have more than one active token per platform.

2. **Org-level only.** All CI secrets are set at the GitHub organization level with visibility "All repositories". No repo-level secrets. No repo-level overrides.

3. **Name contains identity.** Every secret name must identify its owner. No anonymous names like `GH_TOKEN` or `PROJECT_TOKEN`.

4. **GITHUB_TOKEN (auto-generated) is read-only.** The built-in `secrets.GITHUB_TOKEN` is used only for read-only operations (checkout, cache, artifact download). Never use it for releases, project boards, or any mutation.

5. **One secret for all mutations.** A single token per identity handles all write operations on that platform (releases, project boards, tag pushes, etc.). No separate tokens for separate workflows.

6. **Rotation = one action.** Rotating a token means: generate new PAT, update the one org-level secret. All repos pick it up immediately.

7. **Revocation = clean kill.** Deleting the org-level secret revokes all CI write access for that identity across all repos. No repo-level shadows survive.

8. **Future: Authentik-issued.** When Authentik + GitLab CE are deployed, tokens are OIDC-issued per identity. The naming convention stays the same. The issuance method changes.

### Workflow usage

All workflows use `secrets.RUNE_GITHUB_TOKEN` for mutations:

```yaml
# release-please.yml
- uses: googleapis/release-please-action@v4
  with:
    token: ${{ secrets.RUNE_GITHUB_TOKEN }}

# project-board-sync.yml
- uses: actions/add-to-project@v1.0.2
  with:
    github-token: ${{ secrets.RUNE_GITHUB_TOKEN }}
    project-url: https://github.com/orgs/by-openclaw/projects/1
```

### Migration from current state

| Before | After | Action |
|---|---|---|
| `GH_TOKEN` (org) | `RUNE_GITHUB_TOKEN` (org) | Create new, delete old |
| `PROJECT_TOKEN` (org) | `RUNE_GITHUB_TOKEN` (org) | Same token, delete old |
| `GH_TOKEN` (per-repo x5) | deleted | Remove all repo-level secrets |
| `PROJECT_TOKEN` (per-repo x5) | deleted | Remove all repo-level secrets |
| `secrets.GITHUB_TOKEN` in release-please (4 repos) | `secrets.RUNE_GITHUB_TOKEN` | Update workflow files |
| `secrets.GH_TOKEN` in release-please (1 repo) | `secrets.RUNE_GITHUB_TOKEN` | Update workflow file |
| `secrets.PROJECT_TOKEN` in project-board-sync (5 repos) | `secrets.RUNE_GITHUB_TOKEN` | Update workflow files |

## Consequences

- All CI mutations are traceable to a named identity
- Token rotation is a single org-level operation
- No repo-level secret sprawl
- New repos inherit the org secret automatically — zero secret setup per repo
- If yboujraf needs CI under personal identity, add `YBOUJRAF_GITHUB_TOKEN` at org level

## CISO mapping

> Applies only to controls directly relevant to this ADR's scope.
> Do NOT list every ISO control — only those this ADR satisfies, partially satisfies, or gaps.

### ISO/IEC 27001:2022

| Control | Title | Status | Notes |
|---|---|---|---|
| A.5.15 | Access control | ✓ Covered | One token per identity. No shared or anonymous tokens |
| A.5.16 | Identity management | ✓ Covered | Secret name contains owner identity. Audit trail: who triggered what |
| A.5.17 | Authentication information | ✓ Covered | One credential per identity per platform. No duplication |
| A.5.18 | Access rights | ✓ Covered | Token scopes match identity role. by-rune = automation, yboujraf = admin |
| A.8.2 | Privileged access rights | ✓ Covered | GITHUB_TOKEN (auto) = read-only. RUNE_GITHUB_TOKEN = write. Separation of privilege |
| A.8.5 | Secure authentication | ✓ Covered | PATs are bearer tokens with defined scopes. Rotation = one action |

### NIS2 (Directive 2022/2555)

| Article | Requirement | Status | Notes |
|---|---|---|---|
| Art. 21(2)(d) | Supply chain security | ✓ Covered | No anonymous tokens in CI. Every mutation traceable to an identity |
| Art. 21(2)(e) | Network and information system security | ✓ Covered | Org-level only = single point of management. No repo-level shadow tokens |
| Art. 21(2)(j) | Cryptography and encryption | ✓ Covered | Tokens are PATs (bearer). Rotation = new PAT, update one org secret |

## References

- ADR-0010: Naming & Identity Convention
- ADR-0011: Secret Storage Convention
- ADR-0019: Git Workflow and Approval Process
