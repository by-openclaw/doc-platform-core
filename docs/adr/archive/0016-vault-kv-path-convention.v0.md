# ADR-0016: Vault KV Path Convention

- **Status:** Accepted
- **Date:** 2026-04-02
- **Deciders:** @yboujraf

## Context

ADR-0011 established the Phase 1 secret storage convention (JSON files, pre-Vault). Once Vault is deployed (Layer 2 in the platform charter), all secrets migrate to Vault KV v2. A consistent path structure is required so that Ansible, Terraform, and application configs can reference secrets predictably without per-service path negotiation.

## Decision

**Path format:** `secret/{env}/{service}/{key}`

| Segment | Values | Notes |
|---|---|---|
| `env` | `poc`, `prod`, `dev` | Environment tier (ADR-0012) |
| `service` | service name | Matches the VM naming segment, e.g. `authentik`, `gitlab`, `netbox` |
| `key` | key name | Lowercase, hyphen-separated |

**Examples:**

```
secret/poc/authentik/admin-password
secret/poc/gitlab/admin-password
secret/poc/netbox/secret-key
secret/poc/postgresql/superuser-password
secret/poc/traefik/cloudflare-api-token
```

**No root-level paths.** Paths like `secret/authentik-password` are banned. All paths must have the three-segment structure.

**Access control:** Each service has a dedicated Vault policy scoped to its own path prefix (`secret/data/{env}/{service}/*`). No service policy grants cross-service access. Least privilege, always.

## Consequences

- All Ansible vault lookups and Terraform `vault_generic_secret` calls must use this path format.
- Per-service Vault policies must be written and tested before a service reads its first secret.
- Root-level paths from Phase 1 JSON files must not be carried into Vault — they must be re-keyed under the correct path structure.
- Path structure is environment-aware: the same `service/key` exists independently per env. No shared secrets across environments.

## References

- ADR-0011 — Secret Storage Convention (Phase 1 format)
- ADR-0012 — Environment Tier Standard (`poc`, `prod`, `dev` values)
- ADR-0006 §2 — Layer 2 (Vault) in platform charter
- `brainstorming/2026-04-01-vault-kv-standard.md` — Opus draft (source material)
