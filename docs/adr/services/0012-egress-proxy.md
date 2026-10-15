# services/0012 — Egress Proxy: Internet Access for Hosts and People, Authenticated, Listed, Logged

**Status:** Proposed
**Date:** 2026-10-15 (proposed)
**Scope:** How anything inside the platform reaches the internet: hosts (updates, installers, agents) and people (an RDP or desktop session). Replaces the package cache standing in as a proxy for the Windows guests.
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
3. **Lists are data in the inventory**, reviewed by PR like the firewall catalog: `platform_egress_vendor_domains` (derived from the authorized-apps data plus the platform's own vendors), `platform_egress_portals` (the owner's portals), `platform_egress_deny` (categories never allowed: known-bad, ad/tracking, torrents). Default is **deny**. Ports: `443` (CONNECT) and `80`; nothing else unless listed with its reason.
4. **No TLS interception in v1**: the proxy sees hostnames (SNI/CONNECT), not content; the lists are by domain. Interception for content scanning is a later ADR, if ever.
5. **Every request is logged with the identity** (user or host), destination, bytes and verdict → Loki (`job=egress`), retained like the other service logs; a per-person view exists for the owner. Metrics: the pacemaker only (up, requests, denies) — alert on the proxy being down (every egress depends on it).
6. **Windows guests**: WinHTTP (system) and WinINET (session) point at `svc-proxy`; system egress is host-identified, session egress authenticates. The Linux hosts keep `pkgcache` for packages and get `svc-proxy` for everything else (`http_proxy` in the declared place, not in task environments).
7. **Availability:** one instance now; a second behind the services virtual IP when the cluster exists (`infra/0009`) — the proxy is a dependency of patching, so its liveness alert is critical.

## Consequences

- A download from a new portal is a one-line PR to `platform_egress_portals`, not a firewall exception.
- The person who downloaded a file is in the log; a host that reaches an unlisted destination is a finding.
- Plain-HTTP vendor downloads work again (today they cannot); non-443 ports are listed per need.

## Pending decisions (owner)

1. Approve the service (one LXC, Squid, Authentik LDAP for people, IP identification for hosts).
2. The browsing policy for `proxy-users`: allow-list of approved portals (recommended) or allow-all-with-logging for people.
3. Ports beyond 80/443 anyone needs today (none known).
4. Whether the first portal(s) you need now go into the list in the same PR that builds the service — name them and they do.
