# ADR-0011: Secret Storage Convention

**Status:** Accepted
**Date:** 2026-03-31
**Deciders:** @yboujraf

## Context

Platform credentials (NAS API tokens, Proxmox root passwords, GitHub PATs, Discord webhooks) were previously scattered across `.env` files with inconsistent naming and no metadata. When HashiCorp Vault is deployed (Phase 2), credentials must migrate with zero ambiguity — no reformatting, no guessing which env a credential belongs to.

`lib-synology-dsm` ADR-0009 established a JSON schema for local secrets with Vault KV v2 migration paths. This ADR lifts that decision to platform scope: all repos, all services, all environments follow the same convention.

## Decision

### Phase 1 — Flat JSON + .env (current, pre-Vault)

**Storage location:** `workspace/infra/secrets/` on the Rune VM (not git-tracked).

**One JSON file per credential**, following Vault KV v2 structure:

```json
{
  "vault_path": "secret/{scope}/{service}/{env}",
  "description": "Human-readable purpose",
  "access": "rw|ro",
  "owner": "rune|yboujraf",
  "target": "Device — IP:port",
  "fields": {
    "key": "value"
  }
}
```

- `vault_path` — exact Vault KV v2 path for Phase 2 migration: `vault kv put <vault_path> <fields>`
- `description`, `access`, `owner` — human context only (not consumed programmatically)
- `fields` — flat KV, 1:1 with Vault. No nesting beyond one level.
- **No version, no history, no rotation timestamps** — Vault handles that in Phase 2

**File naming convention:** `{scope}-{service}-{env}.json`

| Segment | Values | Notes |
|---|---|---|
| `scope` | `infra`, `ci`, `app`, `net` | Domain of the credential |
| `service` | `proxmox`, `synology`, `github`, `discord`, `gitlab`, ... | Target system |
| `env` | `poc`, `dev`, `test`, `staging`, `acc`, `prod` | Environment tier — **omit if global** |

Examples:
- `infra-proxmox-poc.json` — Proxmox PoC cluster credentials
- `infra-synology-nas.json` — NAS (no env suffix = infrastructure-global, single NAS shared across all envs)
- `ci-github-pat.json` — GitHub PAT (global)

**Companion .env files:** Both `.json` and `.env` variants may exist for the same credential. `.env` files are shell-sourceable for scripts. Keep both in sync.

### Phase 2 — HashiCorp Vault KV v2 (when deployed)

Migration is one command per file:

```
vault kv put secret/{scope}/{service}/{env} key1=value1 key2=value2
```

After migration:
- Delete local JSON/env files — Vault is the single source of truth
- Applications read credentials via Vault API (AppRole auth for service accounts)
- Rotation, versioning, and audit logging handled by Vault natively

### Access rules

| Who | Access | How |
|---|---|---|
| @yboujraf | Full (owner) | Direct file access (Phase 1), Vault admin (Phase 2) |
| Rune agent | Read (runtime) | Sources via environment variables or dotenv — never reads JSON directly |
| Other agents | None | Request via @yboujraf |

### Redaction

- Redaction format: `<REDACTED:{type}>` — types: `password`, `token`, `ssh-pubkey`, `ssh-privkey`, `secret`, `ip-range`
- Never output credential values in chat, docs, memory, CHANGELOG, or any committed file
- Pre-commit hooks: `detect-secrets` on every repo

### Separation from OpenClaw

OpenClaw agent secrets stay in OpenClaw's own config format. `workspace/infra/secrets/*.json` is for platform infrastructure credentials only.

## Consequences

**Positive:**
- Migration to Vault is mechanical — one command per file, no reformatting
- `description` field prevents wrong credential used under pressure (incident response)
- Consistent format across all platform credentials — any new service follows the same pattern
- `vault_path` is pre-computed — no mapping table needed at migration time
- Old `.env` files deprecated — single format going forward

**Negative:**
- JSON not directly sourceable by shell scripts (`.env` was). Scripts must parse or use a helper during Phase 1.
- Local files have no encryption at rest during Phase 1 (Vault solves this in Phase 2)
- Dual format (JSON + .env) during Phase 1 creates sync risk — mitigated by convention, not tooling

## Compliance

- **ISO A.10.1.1** (cryptographic controls policy) — credentials stored with defined schema, encrypted at rest in Phase 2 (Vault)
- **ISO A.9.4.3** (password management) — no plaintext in committed files, redaction enforced
- **ISO A.9.2.4** (management of secret authentication information) — one credential per file, owner tracked, access scoped
- **ISO A.12.4.1** (event logging) — Vault provides full audit trail in Phase 2
- **NIS2 Art.21(2)(d)** (supply chain security) — credential isolation per env prevents cross-env blast radius
- **NIS2 Art.21(2)(e)** (security in network and information systems) — detect-secrets pre-commit hooks prevent credential leakage

## Notes

- Origin: `lib-synology-dsm/docs/adr/0009-secrets-json-schema.md` (repo-scoped) — this ADR is the platform-scoped version
- Full structure reference: `workspace/infra/secrets/README.md`
- Vault migration commands documented in `workspace/infra/secrets/README.md` §Vault migration
- Old `.env` files (`.synology.env`, `.proxmox-nonprod.env`) kept as deprecated — redacted, not deleted
