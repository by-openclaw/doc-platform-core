# naming/0001 — Infrastructure Naming

**Status:** Draft
**Date:** 2026-04-13 (supersedes part of flat ADR-0010, 2026-03-31)
**Scope:** Patterns for physical hosts, VMs, LXCs, DNS names, and related infrastructure assets.
**Related:** `naming/0002-identity`, `services/0003-netbox-cmdb` (future, from flat 0009), `infra/0005-environment-tiers` (future, from flat 0012)

---

## Context

Without a binding infrastructure naming convention, assets accumulate inconsistent names: hostnames diverge from NetBox records, env tiers leak into names that outlive their env, and site/provider information is either absent or embedded so tightly that relocation forces a rename. This ADR defines a stable, NetBox-backed, length-safe pattern.

## Decision

### 1. NetBox is the source of truth

The NetBox `name` field is authoritative for every physical host, VM, and LXC on the platform. Terraform, Ansible, and monitoring **read** NetBox — they never invent or override a name locally. If NetBox and a running host disagree, NetBox wins and the host is reprovisioned or renamed.

Metadata (env, role, site, tenant, primary IP, platform, tags) lives in NetBox fields — **never** in the name itself.

### 2. Length budget — 15 characters

Hostnames and NetBox `name` values must be **≤ 15 characters** to satisfy Windows NetBIOS compatibility. This is forward-compatible with planned Windows 11 / Windows Server hosts (WinRM transport — see `git/0003-configuration §Applicability`). RTBF's broadcast convention uses 16 chars for the same reason (Active Directory). We pick 15 to keep the strictest constraint.

### 3. Pattern

```
{site}-{service}-{seq:02d}
```

- `site` — 2 chars, lowercase — **logical location of the asset** (see §4)
- `service` — 3-7 chars, lowercase — short code for the service or product (see §5)
- `seq` — 2-digit zero-padded index — distinguishes multiple instances of the same service at the same site

All lowercase. Hyphen-separated. No underscores.

**Examples:**

| Asset | Name | Chars |
|---|---|---|
| Proxmox host in Brussels office | `br-pmox-01` | 10 |
| Synology NAS in Brussels office | `br-syno-01` | 10 |
| NetBox VM on Proxmox | `vm-nbox-01` | 10 |
| GitLab CE VM on Proxmox | `vm-glab-01` | 10 |
| Mailcow on Contabo VPS | `cb-mcow-01` | 10 |
| OPNsense running as VM | `vm-opns-01` | 10 |

### 4. Site codes

The **site code answers "where does this asset logically live"**, not "which physical room is it in". Physical rack location is tracked by NetBox `site` / `rack` / `position` fields — the 2-char code above is a broader category.

| Code | Site | Type |
|---|---|---|
| `br` | Brussels office | Physical — BY-SYSTEMS-owned hardware |
| `vm` | Virtual pool on `br` Proxmox | Logical — VMs/LXCs on BY-SYSTEMS-owned hypervisors |
| `cb` | Contabo VPS | Cloud — rented from Contabo |
| `mo` | (Reserved) future Mons office | Reserved physical |
| `ht` | (Reserved) future Hetzner Cloud | Reserved cloud |
| `fp` | (Reserved) future mobile / field pool | Reserved mobile |

**Rules:**
- VMs running on BY-SYSTEMS Proxmox use `vm` — **not** `br` — because they can vMotion, HA-failover, or be restored on a different hypervisor without a rename.
- LXCs use `vm` for the same reason. The VM-vs-LXC distinction lives in NetBox `type`, not in the name.
- Cloud assets use a **per-provider** code (`cb` for Contabo, `ht` reserved for Hetzner, etc.). Migrating between providers is rare and major — a rename then is acceptable and explicit.
- Adding a new site requires updating this table via ADR amendment.

### 4.1 Physical node + PVE storage naming — `poc` token retired (2026-08-30)

- The token `poc` is **retired from all names** (it was never a site code in §4 and reads as an env tier, which `infra/0005` forbids). The live node `srv-proxmox-poc-01` is renamed per this ADR's pattern — target **`br-pmox-01`** (§3's own example) — with `br-pmox-02` for the second physical host. The rename is executed by the **PVE node provisioning workstream** (fresh provisioning / maintenance window), never as ad-hoc surgery.
- **PVE storage IDs** are env- and site-free, `{content}-{backend}`: `data-zfs` (guest disks), `iso-nfs` (ISO/templates), `pbs-s3` (backup datastore). Current `poc-data` / `poc-iso` migrate at the same workstream (storage IDs are referenced by every guest config — rename only with the tooling, in a window).
- ⚠ **Pending decision (fleet naming):** the live fleet uses `{type}-{fullservice}-{seq}` (`vm-adguard-01`, `lxc-traefik-01`) while §3/§4 contract `{site}-{svc}-{seq}` with LXCs under `vm-` (`vm-adgd-01`). Either the ADR is amended to bless the live convention, or guests are renamed at rebuild. Owner: @yboujraf.

### 5. Service codes

Service codes are lowercase, 3-7 chars, chosen to be unambiguous and readable. The **canonical list lives in NetBox** as the platform's service/role vocabulary (see `services/0003-netbox-cmdb` §Roles). This ADR lists examples only — the authoritative list is maintained alongside the NetBox deployment.

Example codes currently in use or planned:

| Service | Code | Service | Code |
|---|---|---|---|
| Proxmox VE | `pmox` | OPNsense | `opns` |
| NetBox | `nbox` | Synology DSM | `syno` |
| GitLab CE | `glab` | GitLab Runner | `glrn` |
| Authentik | `auth` | HashiCorp Vault | `hvlt` |
| Vaultwarden | `vwrd` | Prometheus | `prom` |
| Grafana | `graf` | Loki | `loki` |
| Promtail | `ptail` | PostgreSQL | `pgsql` |
| Redis | `redis` | MariaDB | `mrdb` |
| Mailcow | `mcow` | Traefik | `trfk` |
| NGINX | `nginx` | WireGuard | `wgrd` |
| Pi-hole | `phole` | Unbound | `unbnd` |
| step-ca | `stca` | Wazuh | `wazuh` |
| MinIO | `minio` | AWX | `awx` |

**Rule:** adding a new service = add a row to the NetBox service-code vocabulary; no ADR change required. This ADR documents the **pattern**, not the inventory.

### 6. Asset FQDN vs Service URL

Two DNS names exist per service. They are not interchangeable.

**Asset FQDN** — derived from NetBox `name`, used by ops:
```
{hostname}.{domain}
```
Example: `vm-glab-01.by-research.be`

Used by: SSH, Ansible inventory, monitoring scrape targets, TLS certificate SANs, log/metric source tags.

**Service URL** — operator-chosen alias, user-facing:
```
{service-alias}.{domain}
```
Example: `gitlab.by-research.be`

Used by: browsers, CI pipelines, webhooks, documentation, support channels. Stored in NetBox as a custom field on the VM.

**Why both exist:**
- Reverse proxy is non-optional — TLS, WAF, rate limiting, header rewriting happen at Traefik, not at the app
- One VM can host multiple services (`vm-auth-01` → `auth.by-research.be` + `ldap.by-research.be`)
- Service-URL continuity across VM migrations — the CNAME just flips when the backend moves
- Wildcard TLS cert covers both names with one certificate per DNS zone

### 6.1 DNS record types — dual-stack mandatory

Every asset and every service URL must resolve **both** IPv4 and IPv6. Single-stack deployment is not allowed on this platform (the network supports IPv6 end-to-end via `infra/0004-network-architecture`, future from flat 0015+0032).

| Name type | Record types | Points to |
|---|---|---|
| **Asset FQDN** — physical host, VM, LXC | `A` + `AAAA` | Direct IPv4 + IPv6 of the asset (NetBox `primary_ip4` + `primary_ip6`) |
| **Asset FQDN** — asset behind NAT / no public IPv6 | `A` + `AAAA` (ULA or RFC1918/GUA as applicable) | Internal IPv4 + IPv6 via internal resolver |
| **Service URL** — user-facing via reverse proxy | `CNAME` → `{proxy-asset-fqdn}` | The proxy (e.g. `vm-trfk-01`) which has its own `A` + `AAAA` |
| **Service URL** — direct-served (rare, no proxy) | `A` + `AAAA` | Same IPs as the underlying asset |
| **Reverse DNS** | `PTR` (both `in-addr.arpa` and `ip6.arpa`) | Matching asset FQDN |

**Rules:**
1. **No `A`-only or `AAAA`-only records.** Every asset and every service URL must have both. If IPv6 is not yet provisioned for an asset, that asset is not considered complete — track as a NetBox gap, not a permanent state.
2. **Service URLs are always `CNAME` when possible.** Direct `A` + `AAAA` for a service URL is allowed only when there is no reverse proxy in front (e.g. `ntp.by-research.be` for an NTP server). This must be documented on the NetBox record.
3. **`CNAME` must not be mixed with other records at the same label.** Per RFC 1912, a `CNAME` cannot coexist with `A` / `AAAA` / `MX` / `TXT` at the same name. This rules out `CNAME` at the zone apex.
4. **Reverse DNS (`PTR`) is mandatory for every asset.** Both `in-addr.arpa` (IPv4) and `ip6.arpa` (IPv6) — mail, logs, and some TLS validators need them. `PTR` points back at the asset FQDN, not the service URL.
5. **No hard-coded IPs in application config.** Applications reference FQDNs. IPs are an implementation detail owned by NetBox + DNS.

**Worked example — `gitlab.by-research.be` in prod, dual-stack behind Traefik:**

```
; Asset records — NetBox primary_ip4 + primary_ip6 for each host
vm-trfk-01          IN  A      10.6.224.50
vm-trfk-01          IN  AAAA   2001:db8:6:224::50
vm-glab-01          IN  A      10.6.224.70
vm-glab-01          IN  AAAA   2001:db8:6:224::70

; Service URL — CNAME to the proxy
gitlab              IN  CNAME  vm-trfk-01.by-research.be.

; Reverse — PTR for both families
50.224.6.10.in-addr.arpa.                                    IN PTR  vm-trfk-01.by-research.be.
0.5.0.0.0.0.0.0.0.0.0.0.0.0.0.0.4.2.2.0.6.0.0.0.8.b.d.0.1.0.0.2.ip6.arpa. IN PTR vm-trfk-01.by-research.be.

70.224.6.10.in-addr.arpa.                                    IN PTR  vm-glab-01.by-research.be.
0.7.0.0.0.0.0.0.0.0.0.0.0.0.0.0.4.2.2.0.6.0.0.0.8.b.d.0.1.0.0.2.ip6.arpa. IN PTR vm-glab-01.by-research.be.
```

Browser resolves `gitlab.by-research.be` → CNAME `vm-trfk-01.by-research.be.` → A/AAAA → hits Traefik on v4 **or** v6 → Host-header routed to `vm-glab-01`. Same flow on IPv4 and IPv6 simultaneously.

**Non-prod example (dev):** every record above shifts into `*.dev.by-research.be` zone. CNAME becomes `gitlab.dev.by-research.be → vm-trfk-01.dev.by-research.be.`. Dual-stack rule still applies.

### 7. Environment sub-domain

Env does **not** appear in the NetBox `name`. Env is expressed via the **DNS zone** and the NetBox `env` custom field.

| Env | Asset FQDN | Service URL |
|---|---|---|
| prod    | `vm-glab-01.by-research.be` | `gitlab.by-research.be` |
| dev     | `vm-glab-01.dev.by-research.be` | `gitlab.dev.by-research.be` |
| test    | `vm-glab-01.test.by-research.be` | `gitlab.test.by-research.be` |
| staging | `vm-glab-01.staging.by-research.be` | `gitlab.staging.by-research.be` |
| acc     | `vm-glab-01.acc.by-research.be` | `gitlab.acc.by-research.be` |
| drp     | `vm-glab-01.drp.by-research.be` | `gitlab.drp.by-research.be` |

**Rules:**
- Prod env sub-domain is **omitted** from DNS zones — prod is clean
- All non-prod envs inject an env sub-domain between hostname and domain
- The NetBox `name` field is **the same** across all envs — only the DNS zone changes
- Env migration (e.g. `dev` → `prod`) = update NetBox `env` custom field + move the DNS records to a different zone. **No rename.**
- Tier definitions (`dev`, `test`, `staging`, `acc`, `prod`, `drp`) are in `infra/0005-environment-tiers`. This ADR uses the tokens, does not define them. **`poc` is not a tier** — it was historically conflated with a hostname label and has been dropped from the tier model.

### 8. Wildcard TLS certificates

One wildcard certificate per DNS zone covers both asset FQDNs and service URLs:

```
*.by-research.be         → prod (asset + service)
*.dev.by-research.be     → dev
*.test.by-research.be    → test
*.staging.by-research.be → staging
*.acc.by-research.be     → acc
*.drp.by-research.be     → drp
```

Certificate issuance and renewal strategy is in `security/0004-certificate-strategy` (future).

### 9. `{domain}` — deployment variable

`{domain}` in this ADR is a placeholder. The actual value is set per deployment in the deployment manifest:

```yaml
# deployments/{org}-{env}/deployment.yml
org:
  domain: by-research.be
  env: prod
```

Examples:

| Deployment | `{domain}` resolves to |
|---|---|
| BY-SYSTEMS prod | `by-research.be` |
| BY-SYSTEMS dev | `by-research.be` (same domain, env sub-domain `dev.`) |
| Future client deployment | `client-xyz.com` (different domain entirely) |

No ADR change is required when a domain changes — update the deployment manifest only.

**Reverse DNS:** `.arpa` is used only for PTR records. Forward service and asset FQDNs never use `.arpa`.

### 10. Stability rule — no env / function / resilience in the name

Names must survive relocation, env migration, hypervisor change, and role reassignment without requiring a rename. The following **must not** appear in the hostname:

- Environment tier (`dev`, `test`, `prod`, etc.) — lives in NetBox `env` field and DNS zone
- Functional role beyond the service code (e.g. no `-master-` / `-primary-` / `-worker-`) — lives in NetBox `role` / `tags`
- Redundancy position (`-a`, `-b`, `-main`, `-backup`) — lives in NetBox `tags` or a dedicated custom field
- Physical site when the asset is virtual — lives in NetBox `site` field (the 2-char `site` code in the name is the **logical** site, not the physical rack)

**Security rationale:** reduces information leakage via DNS/hostname enumeration. An attacker scanning DNS should not learn env, role, or failover topology from the name alone.

### 11. Cross-OS note

Pattern is OS-agnostic. Hostnames on Linux (`/etc/hostname`) and Windows (`Computer Name`) follow the same 15-char rule. Ansible transport differs (SSH for Linux, WinRM for Windows — see `git/0003-configuration §Applicability`) but naming is identical.

### 12. Revision triggers

Revise this ADR when:
- A new physical site is added (new 2-char site code)
- A new cloud provider is adopted (new 2-char provider code)
- The 15-char budget is exhausted by a legitimate service (extend cap or compress a service code)
- The broadcast / media module grows enough to need a dual-name pattern (infra name + operator label, à la RTBF Cerebrum)
- NetBox deployment finalizes and the service-code vocabulary is locked in `services/0003-netbox-cmdb`

## Consequences

- Every asset has a stable NetBox record. Rename events become rare and explicit.
- Env migration is a NetBox-field update + DNS-zone change — never a rename.
- Security: hostnames leak no env, role, or redundancy information.
- Windows/Linux parity — the same pattern works for both, future-proofing WinRM-managed hosts.
- Service URL (`gitlab.by-research.be`) and asset FQDN (`vm-glab-01.by-research.be`) are both canonical, documented, and covered by one wildcard cert per zone.
- The canonical service-code list lives in NetBox / `services/0003-netbox-cmdb` — this ADR stays focused on the pattern.

## CISO mapping

| Framework | Controls covered |
|---|---|
| ISO 27001:2022 | A.5.9 (inventory of information and other associated assets), A.8.9 (configuration management), A.8.32 (change management) |
| NIS2 | Art. 21(2)(a) (risk management), Art. 21(2)(c) (business continuity — DR env is first-class) |
