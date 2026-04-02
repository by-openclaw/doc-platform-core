<!--
| Field        | Value            |
|--------------|------------------|
| Created      | 2026-04-02       |
| Last updated | 2026-04-02       |
| Updated by   | Opus             |
-->

# TLS Standard

**Status:** Accepted
**Date:** 2026-04-02
**Owner:** @yboujraf
**ADR:** ADR-0014

## Rule

Every TLS endpoint on the BY-SYSTEMS platform must use a certificate issued by an approved CA (Let's Encrypt or step-ca). Self-signed certificates are strictly forbidden.

## Requirements

1. All services exposing HTTPS must use TLS with a valid, trusted certificate — no exceptions.
2. Public-facing services (reachable externally or via `*.by-systems.be`) MUST use Let's Encrypt via Cloudflare DNS-01 challenge.
3. Internal services (SVC/MGMT VLANs, no public DNS) MUST use step-ca issued certificates.
4. Self-signed certificates are NEVER permitted on this platform.
5. Traefik is the sole TLS termination point for all platform services. Internal service listeners must bind to `localhost` only.
6. The step-ca root CA certificate MUST be distributed to all VMs via cloud-init (`/usr/local/share/ca-certificates/` + `update-ca-certificates`).
7. Certificate renewal MUST be automated (Traefik ACME for LE; step-ca ACME issuer for internal).
8. No service may produce a browser or TLS client warning in its target environment.

## Compliance table

| Requirement | Test | Pass condition |
|---|---|---|
| No self-signed certs | `openssl verify -CAfile <ca> <cert>` | Returns `OK`, CA is LE or step-ca |
| LE for public services | Check Traefik ACME config | `certificatesResolvers` references Cloudflare DNS-01 |
| step-ca for internal | Check Traefik ACME config for internal entrypoints | Resolver references step-ca ACME endpoint |
| Root CA distributed | `ls /usr/local/share/ca-certificates/ | grep by-systems` on any VM | File present |
| Automated renewal | Traefik logs / step-ca logs | No manual renewal events in last 90 days |
| No TLS warnings | Browser / `curl --cacert <root-ca>` | Zero certificate warnings |

## Decision tree

```
Service has public FQDN under *.by-systems.be AND reachable externally?
  YES → Let's Encrypt (Cloudflare DNS-01 via Traefik)
  NO  → step-ca (internal ACME via Traefik)
```

## Override procedure

To override this standard for a specific tool:
1. Create `tools/{tool}/docs/override-tls.md`
2. State: What is different / Why / Compensating control / Reviewed by @yboujraf
3. PR must include override doc before merge is allowed

## References

- [Let's Encrypt documentation](https://letsencrypt.org/docs/)
- [step-ca ACME documentation](https://smallstep.com/docs/step-ca/acme-basics/)
- [Traefik ACME configuration](https://doc.traefik.io/traefik/https/acme/)
- [RFC 8555 — ACME](https://tools.ietf.org/html/rfc8555)
