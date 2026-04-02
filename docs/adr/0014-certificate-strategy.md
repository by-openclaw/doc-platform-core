# ADR-0014: Certificate Strategy

- **Status:** Accepted
- **Date:** 2026-04-02
- **Deciders:** @yboujraf

## Context

The platform serves two classes of TLS endpoints: public-facing services reachable via `*.by-systems.be` and internal services that never leave the platform network. A coherent certificate strategy is required to eliminate self-signed certificates (browser warnings, broken client libraries) and ensure automated renewal with zero manual intervention.

Two CA paths are available: Let's Encrypt (LE) via ACME for public certs, and step-ca for internal PKI. The split must be unambiguous so that any service deployment can determine its CA source without case-by-case decisions.

## Decision

**Public-facing services** (reachable from the internet or via `*.by-systems.be` DNS): LE wildcard certificate via Cloudflare DNS-01 challenge. Traefik terminates TLS. Renewal is automated via Traefik ACME resolver.

**Internal services** (SVC/MGMT VLANs, never public): step-ca signed certificates. step-ca is deployed via cloud-init on a dedicated VM in the SVC zone. Traefik terminates TLS using the step-ca ACME issuer (SCEP or ACME endpoint provided by step-ca).

**Self-signed certificates: NEVER.** No service on this platform uses self-signed TLS.

**Traefik dashboard**: internal only, step-ca cert. Not exposed publicly.

**Decision rule:**
- Service has a public FQDN under `*.by-systems.be` and is reachable externally → LE
- Service is internal only (SVC/MGMT/DMZ, no public DNS) → step-ca

**No-warning guarantee:**
- LE: always valid in browsers and standard clients
- step-ca: valid only if the step-ca root CA is in the system trust store. The cloud-init template **must** inject the root CA into `/usr/local/share/ca-certificates/` and run `update-ca-certificates`. This is non-negotiable — omitting it breaks all internal TLS clients silently.

## Consequences

- Traefik is the single TLS termination point for all services. No per-service cert management.
- step-ca must be deployed before any internal service goes live (Layer 2 in the platform charter).
- Cloud-init template must include root CA distribution to all VMs.
- Wildcard `*.by-systems.be` covers all public subdomains. No per-hostname LE certs.
- Certificate rotation is fully automated. No manual renewal process.
- Split CA model means internal services are not trusted externally (correct and intentional).

## References

- `docs/stack.md` — step-ca and Cloudflare entries
- `docs/naming-convention.md` §6 — FQDN and DNS naming
- ADR-0006 — Layer 2 (step-ca) prerequisite for PKI mount
