# identity/0003 — Machine Credentials (CI Tokens)

**Status:** Draft
**Date:** 2026-04-12 (supersedes flat ADR-0031, 2026-04-05)
**Scope:** Naming and lifecycle of CI/CD tokens used to mutate platform systems.
**Related:** `identity/0001-authentication`, `security/0001-secret-storage`, `naming/0001-naming-convention`

---

## Context

CI workflows need tokens (PATs) to mutate GitHub/GitLab/Vault/OPNsense. Without a naming standard, secrets sprawl (`GH_TOKEN`, `PROJECT_TOKEN`, `GITHUB_TOKEN`) with no owner traceability and duplication across org/repo levels.

## Decision

### Naming convention

Pattern: **`{IDENTITY}_{PLATFORM}_TOKEN`**

- `IDENTITY` = service account or human handle, UPPER_SNAKE_CASE (e.g. `RUNE`, `YBOUJRAF`, `OPUS`)
- `PLATFORM` = target system (`GITHUB`, `GITLAB`, `VAULT`, `OPNSENSE`)
- `TOKEN` = literal suffix

| Secret | Identity | Platform | Purpose |
|---|---|---|---|
| `RUNE_GITHUB_TOKEN` | by-rune | GitHub | CI: releases, project boards, mutations |
| `YBOUJRAF_GITHUB_TOKEN` | yboujraf | GitHub | Manual: repo/org admin |
| `RUNE_OPNSENSE_TOKEN` | by-rune | OPNsense | Integration tests: `key:secret` |
| `RUNE_GITLAB_TOKEN` | by-rune | GitLab CE | Future CI |
| `RUNE_VAULT_TOKEN` | by-rune | Vault | Future secret retrieval (AppRole) |

### PAT vs secret storage

- **PAT:** always created under the `by-rune` GitHub user account. GitHub has no concept of "org-owned PAT".
- **Secret:** where the PAT value is pasted so workflows can read it via `${{ secrets.NAME }}`.

This ADR standardizes the **secret name and storage**, not the PAT lifecycle.

### Rules

1. **One token per identity per platform.** No multiple active tokens.
2. **Storage depends on the CI platform.**
   - **GitHub Free + private repos (current):** repo-level secret `RUNE_GITHUB_TOKEN` in every repo that needs CI mutations. Org-level secrets for private repos are not available on Free plan, and the org will stay on Free (no upgrade planned).
   - **GitLab CE self-hosted (target for all CI):** single group-level CI/CD variable `RUNE_GITLAB_TOKEN` at the top `by-openclaw` group, inherited by all projects. No plan limitation on GitLab CE.
3. **Name contains identity.** No anonymous names (`GH_TOKEN`, `PROJECT_TOKEN`). The name is identical across all repos/projects so workflow YAML is portable.
4. **`GITHUB_TOKEN` (built-in) is read-only.** Use only for checkout/cache/artifact. Never for mutations.
5. **One secret for all mutations.** A single token per identity handles every write op on that platform.
6. **Rotation.**
   - GitHub (repo-level): new PAT → update the secret in every repo via `gh secret set` loop (runbook).
   - GitLab CE (group-level): update one group variable, all projects inherit.
7. **Revocation.** Revoke the PAT at the source (GitHub/GitLab); stale secrets become inert immediately.
8. **Future: Authentik-issued.** When Authentik is deployed, tokens become OIDC-issued per identity. Naming stays the same; issuance changes.

### Workflow usage

```yaml
# release-please.yml
- uses: googleapis/release-please-action@v4
  with:
    token: ${{ secrets.RUNE_GITHUB_TOKEN }}

# project-board-sync.yml
- uses: actions/add-to-project@v1.0.2
  with:
    github-token: ${{ secrets.RUNE_GITHUB_TOKEN }}
```

## Consequences

- All CI mutations traceable to a named identity
- **Current state (GitHub Free, private repos):** token rotation updates N repos — automated via `gh secret set` loop
- **Target state (GitLab CE self-hosted):** all CI migrates to GitLab CE. `RUNE_GITLAB_TOKEN` lives as a single group variable at `by-openclaw` top group from day one. Rotation = one action.
- **GitHub remains** as the public-facing remote and mirror for open-source repos. No upgrade to Team plan is planned.
- Every new private repo needing CI mutations on GitHub must have `RUNE_GITHUB_TOKEN` set at creation (add to repo bootstrap checklist)
- If yboujraf needs CI under personal identity, add `YBOUJRAF_{PLATFORM}_TOKEN` following the same storage rule

## Known gaps (2026-04-12)

- `ansible-platform` repo is missing `RUNE_GITHUB_TOKEN` — must be added before any CI workflow requires mutations there

## CISO mapping

| Framework | Controls covered |
|---|---|
| ISO 27001:2022 | A.5.15, A.5.16, A.5.17, A.5.18, A.8.2, A.8.5 |
| NIS2 | Art. 21(2)(d), 21(2)(e), 21(2)(j) |
