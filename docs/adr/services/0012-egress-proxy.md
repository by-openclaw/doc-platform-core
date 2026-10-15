# services/0012 — Egress Proxy: Internet Access for Hosts and People, Authenticated, Listed, Logged

**Status:** Proposed
**Date:** 2026-10-15 (proposed)
**Scope:** How anything inside the platform reaches the internet: hosts (updates, installers, agents) and people (an RDP or desktop session) — standing allow-lists for vendors, and **time-boxed slots requested through a portal** for everything else. Replaces the package cache standing in as a proxy for the Windows guests.
**Related:** `infra/0011-windows-authorized-applications` (the vendor sources it allows), `security/0001` (credentials in Vault), `identity/0004` (people and groups), `services/0006-firewall-services` (the catalog that makes it the only way out), `infra/0006-logging`.

---

## Context

Linux hosts fetch packages through the package cache (apt-cacher-ng, `pkgcache`). The Windows guests, by their firewall contract (default-deny outbound), were pointed at the same service for every egress — and it is a *package cache*, not a proxy: it tunnels HTTPS (CONNECT to 443) and refuses plain HTTP or any other port (measured 2026-10-15 from a Windows guest: HTTPS file download 200, GitHub 200, plain HTTP refused, port 8443 refused). The owner's immediate need — download files from a vendor portal during an RDP session — works only when that portal is HTTPS on 443, and no one can say afterwards *who* downloaded *what*: the cache has no notion of a person.

There is also no list: nothing says which destinations a host may reach or a person may browse. "Authorized applications" (`infra/0011`) names the software; nothing names its sources.

## Decision

1. **One egress proxy service**, `svc-proxy` (Squid in the platform's container pattern, one LXC in the services zone, dual-stack), is **the only path to the internet** for every host and every person session. The firewall catalog allows `:80/:443` to the internet from that LXC alone; every other host's internet rule becomes "to the proxy". The package cache keeps its job (apt/pip caching) and chains to the proxy for its own fetches.
2. **Two kinds of clients, two policies:**
   - **Hosts** (machine egress: Windows Update, Defender, Wazuh, agents, installers run by Ansible): identified by **source address** (the platform's zones), no credential, allowed to the **vendor allow-list** only — the sources of the software the platform declares (`infra/0011`'s installers, Microsoft update endpoints, Debian/Ubuntu mirrors, the agents' vendors). A host cannot browse.
   - **People** (an RDP session on a Windows guest, a desktop): **authenticate with their Authentik account** — Squid's basic auth against Authentik's LDAP outpost (the mandatory-SSO rule's LDAP tier; Squid has no OIDC) — and are allowed to the **browsing allow-list** of their group: `proxy-users` (the owner-approved portals and vendor sites) and `proxy-admins` (everything except the deny-list). Per-session credentials never touch Windows: the browser asks once per session.
3. **Lists are data in the inventory**, reviewed by PR like the firewall catalog — with one category that is not optional: **certificate revocation endpoints (CRL/OCSP) over plain HTTP** for every CA in use. Measured 2026-10-15 from a Windows guest behind the package cache: `github.com` fails the TLS handshake with `CRYPT_E_REVOCATION_OFFLINE` because Windows fetches the CA's CRL over HTTP and the cache refuses plain HTTP — an HTTPS-only proxy silently breaks a share of the web for every Windows client. The egress proxy allows port 80 to the listed CA endpoints for all clients, before any other list.
   The other lists: `platform_egress_vendor_domains` (derived from the authorized-apps data plus the platform's own vendors), `platform_egress_portals` (the owner's portals), `platform_egress_deny` (categories never allowed: known-bad, ad/tracking, torrents). Default is **deny**. Ports: `443` (CONNECT) and `80`; nothing else unless listed with its reason.
4. **No TLS interception in v1**: the proxy sees hostnames (SNI/CONNECT), not content; the lists are by domain. Interception for content scanning is a later ADR, if ever.
5. **Every request is logged with the identity** (user or host), destination, bytes and verdict → Loki (`job=egress`), retained like the other service logs; a per-person view exists for the owner. Metrics: the pacemaker only (up, requests, denies) — alert on the proxy being down (every egress depends on it).
6. **Windows guests**: WinHTTP (system) and WinINET (session) point at `svc-proxy`; system egress is host-identified, session egress authenticates. The Linux hosts keep `pkgcache` for packages and get `svc-proxy` for everything else (`http_proxy` in the declared place, not in task environments).
7. **Availability:** one instance now; a second behind the services virtual IP when the cluster exists (`infra/0009`) — the proxy is a dependency of patching, so its liveness alert is critical.

## Internet by request (the portal)

The owner's model (2026-10-15): nobody gets standing internet; a person **requests a slot** — a destination (FQDN, IP or CIDR, with its ports), a purpose (a license update, a vendor download) and a duration — and the platform opens exactly that, for that time, for that requester, and logs it.

1. **The portal** is a module of the platform's admin portal, behind Authentik (the person is who signed in; no form field asks who they are). The form: destination, ports (80/443 default), purpose (free text, kept in the audit), duration from fixed slots (1 h, 4 h, 8 h; the maximum is a declared value), scope (my session only / the host I name).
2. **Approval** is group-based: a `proxy-admins` member's request is active on submit; a `proxy-users` request waits for an admin's approval in the same portal (the admin sees who, what, why). Every state change notifies the platform's channel.
3. **Enforcement without reloads:** Squid consults the slots through an external ACL helper (requester or source host, destination, port, now < expiry) on every request — a slot is live the second it is approved and dead the second it expires; nothing is rewritten, nothing restarts. The permanent allow-lists of §3 stay for the vendor sources; slots cover everything else.
4. **Hosts that cannot speak to a proxy** (a licensing agent dialling out directly): a slot may ask for a **firewall slot** instead — the same request creates a time-boxed rule through the firewall catalog (`lib-opnsense`, the platform's firewall-as-code) and removes it at expiry. Firewall slots are v2: the catalog needs an expiry field first (its own change, same ADR family as `services/0008`).
5. **Audit:** a slot is a record — requester, approver, destination, purpose, start, expiry, bytes used — written to Loki (`job=egress`, `kind=slot`) and visible in the portal's history; the register names the portal as the control for "internet access by exception".

## Consequences

- A download from a new portal is a one-line PR to `platform_egress_portals`, not a firewall exception.
- The person who downloaded a file is in the log; a host that reaches an unlisted destination is a finding.
- Plain-HTTP vendor downloads work again (today they cannot); non-443 ports are listed per need.

## Pending decisions (owner)

1. Approve the service (one LXC, Squid, Authentik LDAP for people, IP identification for hosts) **with the request portal** as its front door.
2. Approval model: `proxy-admins` self-approve, `proxy-users` need an admin (recommended) — or self-service for everyone with the audit as the control.
3. Slot durations and the maximum (proposed 1 h / 4 h / 8 h, max 24 h).
4. Portal placement: a module of the admin portal (recommended) or a standalone page.
5. The portal(s) you need now — name them (FQDN, port) and they ship as the first standing entries with the service; until the service exists, a time-boxed catalog PR can open one destination for one guest.
