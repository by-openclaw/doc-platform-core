<!--
| Field        | Value            |
|--------------|------------------|
| Created      | 2026-04-02       |
| Last updated | 2026-04-02       |
| Updated by   | Opus             |
-->

# Credential Standard

**Status:** Accepted
**Date:** 2026-04-02
**Owner:** @yboujraf
**ADR:** ADR-0011, ADR-0016

## Rule

All platform credentials (infrastructure, service accounts, automation tokens) MUST be stored in HashiCorp Vault (Phase 2 onwards) under the canonical KV path structure. No credential may be stored in plaintext in any file, environment variable, or log. Phase 1 pre-Vault credentials follow the JSON schema defined in ADR-0011.

## Requirements

1. No credential may be stored in plaintext in any committed file — ever.
2. All infrastructure credentials MUST follow the Vault KV path format: `secret/{env}/{service}/{key}`.
3. Phase 1 (pre-Vault): credentials stored as JSON files in `workspace/infra/secrets/` (not git-tracked), following the schema: `{ vault_path, description, access, owner, target, fields }`.
4. Phase 2 (Vault deployed): all credentials migrated to Vault KV v2 using the exact `vault_path` from Phase 1 JSON.
5. Each service MUST have a dedicated Vault policy scoped to its own path prefix — no cross-service access.
6. Credentials MUST be per-environment: `poc` credentials cannot access `prod` resources.
7. Service account credentials are managed under `secret/{env}/{service}/` — application credentials under `secret/app/{service}/`.
8. Rotation: [OWNER TO DEFINE: rotation schedule and ownership per credential class].
9. This standard governs infra/platform credentials only. Library-level credential handling (e.g., how lib-synology-dsm passes credentials to its client) is governed by lib-scoped ADRs.

## Vault KV path structure

| Segment | Values | Notes |
|---|---|---|
| `env` | `poc`, `dev`, `test`, `staging`, `acc`, `prod` | Environment tier — never omit |
| `service` | service name | Matches VM naming segment (e.g. `authentik`, `gitlab`) |
| `key` | key name | Lowercase, hyphen-separated |

**Examples:**
```
secret/poc/authentik/admin-password
secret/poc/gitlab/admin-password
secret/poc/netbox/secret-key
secret/poc/postgresql/superuser-password
secret/poc/traefik/cloudflare-api-token
```

## Compliance table

| Requirement | Test | Pass condition |
|---|---|---|
| No plaintext creds in git | `git grep -i "password\|secret\|token" --` on all repos | Zero matches in committed files (redacted stubs only) |
| Phase 1 JSON schema valid | Validate each secrets JSON | All fields present: `vault_path`, `description`, `access`, `owner`, `target`, `fields` |
| Path format correct | Audit Vault KV paths | All paths match `secret/{env}/{service}/{key}` |
| Per-service Vault policy | List Vault policies | One policy per service, scoped to `secret/data/{env}/{service}/*` |
| Env-separated credentials | Cross-check poc vs prod paths | No shared credentials across environments |

## Override procedure

To override this standard for a specific tool:
1. Create `tools/{tool}/docs/override-credentials.md`
2. State: What is different / Why / Compensating control / Reviewed by @yboujraf
3. PR must include override doc before merge is allowed

## References

- [HashiCorp Vault KV v2 documentation](https://developer.hashicorp.com/vault/docs/secrets/kv/kv-v2)
- [Vault policies](https://developer.hashicorp.com/vault/docs/concepts/policies)
- [ISO 27001:2022 A.8.12 — Prevention of data leakage](https://www.iso.org/standard/27001)
