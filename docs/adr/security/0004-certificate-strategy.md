# security/0004 — Certificate Strategy

**Status:** Draft
**Date:** 2026-04-13 (supersedes flat ADR-0014, 2026-04-02)
**Scope:** TLS certificate issuance — which CA issues which cert, for which endpoint, with what automation. No self-signed certificates anywhere.
**Related:** `naming/0001-infra §9` (the `{domain}` deployment variable), `security/0001-secret-storage`, `security/0003-hardening`, `infra/0002-platform-charter` (future)

---

## Context

The platform serves two classes of TLS endpoints: **public-facing** services reachable via a real domain (e.g. `*.by-research.be`) and **internal** services that never leave the platform network. A coherent certificate strategy is required to eliminate self-signed certificates — which cause browser warnings, break client libraries, and force operators to click through trust prompts — and to ensure automated renewal with zero manual intervention.

Two CA paths are available: Let's Encrypt (LE) via ACME for public certs, and step-ca for internal PKI. The split between them must be unambiguous so that any service deployment determines its CA source mechanically, without case-by-case decisions.

## Decision

### The two CAs

| CA | Purpose | Challenge method | Renewal |
|---|---|---|---|
| **Let's Encrypt (LE)** | Public-facing services reachable from the internet | DNS-01 via Cloudflare (wildcard support) | Traefik ACME resolver, fully automated |
| **step-ca** (self-hosted) | Internal services on SVC / MGMT / DMZ VLANs, never public | ACME endpoint provided by step-ca | Traefik ACME resolver (same mechanism, different directory URL) |

**Self-signed certificates are forbidden.** No service on this platform uses self-signed TLS. Every TLS endpoint resolves to a valid, trusted chain — either LE's public root or step-ca's internal root injected into the VM trust store.

### Decision rule — which CA to use

Apply in priority order:

1. **Domain is IANA-reserved** (`example.com`, `test`, `localhost`, `lab`, etc.) → **step-ca always**. LE cannot and will not issue for IANA-reserved domains regardless of the service's network topology.
2. **Service is internal only** (reachable only from SVC / MGMT / DMZ VLANs, no public DNS record) → **step-ca**.
3. **Service has a public FQDN and is reachable externally** → **LE**.

No other rule applies. No per-service exception. If a service cannot decide its CA from these three rules, its architecture is wrong — fix the architecture, not the cert strategy.

### Env tier → CA mapping (practical reference)

| Env tier | Typical domain | CA |
|---|---|---|
| `dev`, `test` | IANA-reserved (`example.com`) or internal-only zone | step-ca |
| `staging`, `acc` | Real domain, internal exposure only | step-ca (default) or LE (if publicly reachable) |
| `prod` | Real domain, publicly reachable | LE |
| `drp` | Same as prod | LE (must be issuable during DR — Cloudflare DNS-01 does not depend on primary site availability) |

`{domain}` is set per deployment in the deployment manifest (see `naming/0001-infra §9`).

### Traefik as the TLS termination point

- **Traefik is the single TLS termination point** for all services. No per-service cert management, no Apache/Nginx cert configuration, no application-level TLS.
- Traefik's ACME resolver is configured with two issuers: `letsencrypt` (LE prod directory) and `stepca` (step-ca directory URL).
- Each router in Traefik declares which issuer it uses via a static config file or a Docker label (`traefik.http.routers.<name>.tls.certresolver=stepca`).
- Internal UIs (Traefik dashboard, Grafana admin, etc.) always use `stepca`. Public services always use `letsencrypt`.

### step-ca root CA distribution — non-negotiable

step-ca certificates are only valid on clients whose system trust store contains the step-ca root CA. The cloud-init template for every VM **must**:

1. Write the step-ca root CA to `/usr/local/share/ca-certificates/stepca.crt`
2. Run `update-ca-certificates` to rebuild the system bundle
3. Restart services that cache the trust store at startup (systemd `ca-certificates-updates.target` hook)

**Omitting this step breaks all internal TLS clients silently** — curl fails, Python `requests` fails, Ansible `uri` module fails, browsers show a warning. Every failure mode is a support ticket.

This is enforced in `ansible-platform/roles/ca-trust` (new, to be created as part of the security scope implementation). Every playbook that provisions a VM includes this role as a pre-dependency.

### Root CA lifecycle

- **LE root** — distributed by the OS / container base image. No action required.
- **step-ca root** — generated once at step-ca deployment, rotation is a major event (new root = re-trust every VM). Intermediate CA rotation is automated.
- **step-ca root key** storage: sealed in HashiCorp Vault (`secret/prod/stepca/root-ca-key`), never on the step-ca host filesystem after initial setup. See `security/0001-secret-storage`.
- **Root CA expiry** is monitored; alert 90 days before expiry; rotation runbook in `platform-setup/runbooks/stepca-root-rotation.md` (future).

### Wildcard strategy

- **One wildcard cert per DNS zone**, covering both asset FQDNs and service URLs (see `naming/0001-infra §8`).
- LE wildcards require DNS-01 challenge — Cloudflare API token is stored in Vault at `secret/prod/traefik/cloudflare-api-token`.
- step-ca wildcards are issued via ACME with HTTP-01 or DNS-01, internal-only resolver. No per-hostname cert generation — Traefik reuses the zone wildcard.

### Cross-OS note

Certificate strategy is **transport-agnostic** — it applies identically to Linux hosts (current) and Windows 11 / Windows Server hosts (future, WinRM transport — see `git/0003-configuration §11`). The difference is in root CA distribution:

- **Linux:** `/usr/local/share/ca-certificates/` + `update-ca-certificates`
- **Windows:** `certutil -addstore Root ...` via a dedicated Ansible task (future, added with the Windows branch of `ca-trust` role)

The CA split (LE public / step-ca internal), the decision rule, and the Traefik termination model are identical on both OSes.

## Consequences

- **Single TLS termination point** (Traefik) means cert rotation is one process, not N
- **Zero manual cert management** — ACME handles everything, renewal failures alert to Discord
- **step-ca deployment is a Layer 2 prerequisite** (future `infra/0002-platform-charter`). No internal service goes live before step-ca is running and its root CA is in every VM's trust store.
- **Root CA distribution via cloud-init is enforced by role** — a VM without the role cannot reach internal TLS services
- **Split trust model is correct and intentional** — internal services are not trusted externally. Leaking an internal service URL does not expose it to unauthenticated scanners.
- **Wildcard-per-zone reduces cert volume** — 1 cert covers `vm-glab-01.by-research.be`, `vm-nbox-01.by-research.be`, `gitlab.by-research.be`, `netbox.by-research.be`, …
- **Cloudflare dependency for LE** — if Cloudflare API is unreachable for more than cert expiry - 30 days, LE renewal fails. Accepted risk; mitigated by 90-day cert lifetime and automated alerts.

## Revision triggers

Revise when:
- A third CA is introduced (e.g. DigiCert for EV certificates, or a client-specific private CA)
- LE is replaced (unlikely — industry standard) or deprecates DNS-01
- step-ca is replaced by a different internal PKI (HashiCorp Vault PKI engine, cfssl, etc.)
- Cloudflare is replaced as the DNS provider for LE DNS-01 challenges
- Wildcard issuance becomes insufficient (e.g. a per-service cert is required for pinning or compliance)
- Client-auth certs (mTLS) are introduced at scale — current scope is server certs only
- Windows hosts are introduced and the `ca-trust` role gains a Windows branch

## CISO mapping

| Framework | Controls covered |
|---|---|
| ISO 27001:2022 | A.5.14 (information transfer — all service communication uses TLS), A.8.23 (web filtering — no-warning guarantee), A.8.24 (use of cryptography — LE + step-ca, no self-signed) |
| NIS2 | Art. 21(2)(e) (network and information systems security — automated renewal eliminates manual rotation risk) |
| GDPR | Art. 32(1)(a) (encryption of personal data in transit) |
