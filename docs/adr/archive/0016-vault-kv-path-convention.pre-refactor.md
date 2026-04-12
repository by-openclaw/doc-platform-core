# ADR-0016: Vault KV Path Convention

- **Status:** Accepted
- **Date:** 2026-04-02
- **Deciders:** @yboujraf

## Context

The platform secret storage convention established the Phase 1 format (JSON files, pre-Vault). Once Vault is deployed (Layer 2 in the platform charter), all secrets migrate to Vault KV v2. A consistent path structure is required so that Ansible, Terraform, and application configs can reference secrets predictably without per-service path negotiation.

## Decision

**Path format:** `secret/{env}/{service}/{key}`

| Segment | Values | Notes |
|---|---|---|
| `env` | `poc`, `prod`, `dev` | Environment tier — six tiers defined in environment tier standard |
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

## CISO mapping

> Applies only to controls directly relevant to this ADR's scope.

### ISO/IEC 27001:2022

| Control | Title | Status | Notes |
|---|---|---|---|
| A.8.24 | Use of cryptography | ⚠ Partial | Vault KV v2 path structure defined; Vault not yet deployed |
| A.5.17 | Authentication information | ✓ Covered | Per-service policy scoped to own path prefix; no cross-service access |
| A.8.12 | Prevention of data leakage | ✓ Covered | Structured paths enforce env separation; root-level paths are banned |

### NIS2 (Directive 2022/2555)

| Article | Requirement | Status | Notes |
|---|---|---|---|
| Art. 21(2)(d) | Supply chain security | ✓ Covered | Env-separated credential paths prevent cross-env blast radius |

### GDPR (Regulation 2016/679)

Not applicable — this ADR governs infrastructure credential path structure, not personal data.

## Consequences

- All Ansible vault lookups and Terraform `vault_generic_secret` calls must use this path format.
- Per-service Vault policies must be written and tested before a service reads its first secret.
- Root-level paths from Phase 1 JSON files must not be carried into Vault — they must be re-keyed under the correct path structure.
- Path structure is environment-aware: the same `service/key` exists independently per env. No shared secrets across environments.

## References

- Secret storage convention defines the Phase 1 JSON format and Vault migration path
- Environment tier standard defines the six tiers used in path segments
- Platform charter defines Vault as a Layer 2 dependency before secrets management is operational
- `brainstorming/2026-04-01-vault-kv-standard.md` — Opus draft (source material)
