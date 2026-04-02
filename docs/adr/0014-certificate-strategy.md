# ADR-0014: Certificate Strategy

- **Status:** Draft
- **Date:** 2026-04-02
- **Deciders:** @yboujraf

## Context

The platform uses both public TLS certificates (Let's Encrypt via Cloudflare DNS-01 for `*.by-systems.be`) and an internal CA (step-ca for internal service mesh). A formal decision is needed to define how these two systems coexist, which services use which CA, and how certificate provisioning integrates with Traefik.

## Decision

TODO: pending @yboujraf review

## Consequences

TODO: pending decision

## References

- `docs/stack.md` — step-ca and Cloudflare entries
- `docs/naming-convention.md` §6 — FQDN and DNS naming
- ADR-0006 — Layer 2 (Vault) prerequisite for PKI mount
</content>