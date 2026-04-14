# BY-SYSTEMS PoC Platform — Architecture Scope v1
> Status: **Draft — pending Opus audit**
> Author: Rune (brainstorming session 2026-04-01 with @yboujraf)
> Not yet tested. Commands/configs are design intent, not verified.

---

## 1. Scope & Purpose

This document defines the architecture decisions, tool selections, and design rationale for the BY-SYSTEMS PoC platform. It covers network services, time synchronization, DNS, and infrastructure routing as discussed in the 2026-04-01 brainstorming session.

**Goals:**
- Clean, non-redundant service stack (no duplicate tools doing the same job)
- CISO-grade defaults: encrypted DNS, forced NTP discipline, VLAN isolation
- OPNsense as the single control plane for DNS, NTP, firewall, VPN
- Proxmox as the hypervisor — VMs for specialized services only where justified

**Out of scope for this document:**
- PTP design and validation
- Broadcast-grade timing distribution
- Arista 7020/7060 PTP downstream behavior

PTP exists in production and works there; it is intentionally parked and excluded from this PoC scope to keep the current design focused.

---

## 2. Platform Components — Final Decisions

### 2.1 Firewall / Network Control Plane
**Tool: OPNsense (VM on Proxmox)**
- Replaces: pfSense
- Handles: WAN failover, VLAN routing, DNS (Unbound), NTP (ntpd), DoT upstream, firewall, VPN

**WAN connectivity:**
- ISP interfaces via Proxmox Linux bridge (`vmbr`) — tested, works fine
- No PCIe passthrough required for WAN NICs
- OPNsense sees virtio interfaces bridged to physical ISP-facing ports

**Pushed back on:** Separate Pi-hole VM, separate Chrony VM — both redundant when OPNsense Unbound + ntpd cover the same functionality natively.

---

### 2.2 DNS Architecture
**Tool: OPNsense Unbound DNS Resolver**
- No Pi-hole. Unbound has equivalent DNSBL + local records + DoT.

**Domain split:**
| Zone | Example FQDN | DNS authority | Resolves to |
|---|---|---|---|
| Public services | `gitlab.poc.by-systems.be` | Cloudflare | Telenet public IP |
| Internal-only | `vault.int.poc.by-systems.be` | OPNsense Unbound | `10.x.x.x` internal |
| WireGuard/VPN | `wg.poc.by-systems.be` | Cloudflare | Telenet public IP |

**Split DNS mechanism:**
- External clients → Cloudflare resolves public zone only
- Internal clients → DHCP hands out OPNsense IP as resolver → Unbound serves local overrides for `int.poc.by-systems.be`
- `int.poc.by-systems.be` zone never exposed externally

**TLS strategy:**
- Public services → Let's Encrypt HTTP challenge (standard)
- Internal services → Let's Encrypt **DNS challenge** via Cloudflare plugin on Traefik → wildcard `*.int.poc.by-systems.be` — one cert, all internal services, no cert warnings

**DNS over TLS (DoT):**
- Configured once in `Services → Unbound → DNS over TLS`
- Upstream: Cloudflare 1.1.1.1:853 + 1.0.0.1:853
- All VLANs that use Unbound automatically benefit — no per-VLAN config

**Forced DNS (prevent bypass):**
- Floating firewall rule — direction: OUT — WAN interface
- Block UDP+TCP port 53 to destination ≠ OPNsense IP
- One rule covers all VLANs — no per-VLAN repetition
- Scope: `dest != OPNsense_IP` — does NOT break inter-VLAN traffic (lesson from pfSense migration)

**pfBlocker equivalent:**
- Phase 1: `Services → Unbound → Blocklists` (Hagezi / Steven Black lists) — ad/malware DNSBL
- Phase 2 (optional): Zenarmor (os-sensei) for per-VLAN L7 policies if needed

---

### 2.3 Reverse Proxy / Ingress
**Tool: Traefik**
- Manages multiple domains — public + internal — from single instance
- Routing by `Host()` rule — one Traefik, many domains, mixed public/internal

**Example rules:**
```
# Public
rule: "Host(`gitlab.poc.by-systems.be`)"

# Internal only
rule: "Host(`vault.int.poc.by-systems.be`)"
```

**TLS:**
- Public: HTTP challenge (auto via Let's Encrypt)
- Internal: DNS challenge (Cloudflare) → wildcard cert `*.int.poc.by-systems.be`

---

### 2.4 NTP
**Tool: OPNsense ntpd (built-in)**
- No separate Chrony VM
- OPNsense syncs to upstream NTP pools (stratum 1) → serves all VLANs as stratum 2
- DHCP hands out OPNsense IP as NTP server — all clients auto-configured
- Proxmox host configured to use OPNsense as NTP source (`/etc/chrony.conf`)

**Forced NTP (prevent bypass):**
- Floating firewall rule — block UDP port 123 to destination ≠ OPNsense IP
- Same pattern as DNS rule — one floating rule, all VLANs

**Accuracy:** ~1–10ms. Sufficient for Proxmox, VMs, k3s, Vault, Authentik, log correlation.

---

### 2.5 ISP / WAN Routing
**Method: Proxmox Linux bridge (vmbr)**
- Confirmed approach — vmbr works fine for OPNsense WAN
- No PCIe passthrough needed for ISP interfaces
- PPPoE (Proximus) + Telenet via bridge → OPNsense handles protocol termination

---

## 3. What is NOT built yet

| Item | Status | Next step |
|---|---|---|
| OPNsense Unbound split DNS | Not configured | Manual config on OPNsense |
| Traefik multi-domain + DNS challenge | Not deployed | Terraform + Helm/compose |
| Floating firewall rules (DNS/NTP) | Not applied | OPNsense config |

---

## 4. Key Decisions & Pushback Summary

| Decision | Rejected alternative | Reason rejected |
|---|---|---|
| OPNsense Unbound for DNS | Pi-hole | Redundant — Unbound is a superset |
| OPNsense ntpd for NTP | Chrony VM | Redundant — ntpd built-in, same result |
| vmbr for ISP/WAN | PCIe passthrough | Passthrough not needed — vmbr confirmed working |
| Central DNS/NTP enforcement policy | Per-VLAN one-off rules everywhere | Keep policy consistent and maintenance low |
| Start strict, add exceptions | Replicate pfSense rule complexity | Build clean, add exceptions with justification |

---

## 5. Open Questions (for audit)

1. Is `int.poc.by-systems.be` the right internal subdomain convention, or should it be `internal.` / `lan.` / something else?
2. Zenarmor licensing — free tier sufficient for PoC or does it gate needed features?
3. DNS challenge for internal TLS — Cloudflare API token scoped correctly (zone edit only for `by-systems.be`)?

---

## 6. Audit Request for Claude Opus

**Please audit this document for:**

1. **Architectural correctness** — any technical errors, wrong assumptions, missing dependencies
2. **Security gaps** — what's not covered, what's dangerously underspecified
3. **Simplification opportunities** — anything overcomplicated for a PoC
4. **Missing components** — what else is typically needed in this stack that isn't mentioned
5. **Sequencing** — is the build order implied here sensible? What should be done first?
6. **OPNsense-specific** — any Unbound/ntpd/firewall gotchas not covered
7. **Split DNS** — any edge cases with the `.int.poc.by-systems.be` approach

Be blunt. Flag anything wrong or incomplete. This is a design review, not a rubber stamp.
