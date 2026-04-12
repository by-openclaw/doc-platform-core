# PoC Platform Design — BY-SYSTEMS
> Status: Phase 1 — Install Only  
> Last updated: 2026-04-01  
> Owner: yboujraf / Rune

---

## Scope

### In scope
- OPNsense installation (VM on srv-proxmox-poc-01)
- Proxmox SDN (virtual only, no physical switch dependency)
- Terraform (dedicated VM)
- step-ca (internal CA)
- Traefik (reverse proxy, DMZ)
- NetBox (IPAM + documentation)
- Storage decision (TBD)

### Out of scope (explicitly)
- PTP — works in prod, not needed in PoC
- WAN failover — two ISPs exist, no auto-failover design yet
- Physical switch config — SDN first
- Firewall policy — phase 2
- DNS/NTP enforcement rules — phase 2

---

## DNS Naming Convention

Source of truth: **ADR-0010** (naming-and-identity-convention), **docs/naming-convention.md §6**

### Pattern
```
{service}-{env}-{seq:02d}.by-systems.be
```
Env is carried in the **hostname**. Zone is always `by-systems.be`. No env-level subdomain.

### VM / hostname naming (per ADR-0010)
```
vm-{service}-{env}-{seq:02d}
```

### FQDN examples
| Machine name | FQDN |
|---|---|
| `vm-netbox-poc-01` | `netbox-poc-01.by-systems.be` |
| `vm-traefik-poc-01` | `traefik-poc-01.by-systems.be` |
| `vm-stepcca-poc-01` | `stepcca-poc-01.by-systems.be` |
| `vm-opnsense-poc-01` | `opnsense-poc-01.by-systems.be` |
| `vm-vaultwarden-poc-01` | `vaultwarden-poc-01.by-systems.be` |
| `vm-authentik-poc-01` | `authentik-poc-01.by-systems.be` |

### Split DNS
- Unbound resolves `*.by-systems.be` internally
- Cloudflare has only records you explicitly publish
- Same registered domain, no fake TLDs, no reserved TLDs

### TLS
- **Internal:** per-hostname cert issued by step-ca
- **Public (if exposed):** LE via DNS-01 challenge (Cloudflare)
- No browser warnings for internal services — trust root distributed to clients
- No concession on PKI: step-ca only, no raw self-signed

### Internal CA hierarchy
- Root CA: step-ca (offline or semi-offline)
- Intermediate CA: online, issues leaf certs
- Trust root distributed to all managed clients

---

## WAN / ISP

| Link | Provider | VLAN | Type | Role |
|---|---|---|---|---|
| WAN1 | Proximus | 10 | PPPoE | Secondary |
| WAN2 | Telenet | 999 | Static | **TBD** — needs testing |

- **Primary WAN:** OOB prod (10.6.224.x) — confirmed working
- **Secondary WAN:** Proximus PPPoE
- **Telenet:** Static IP available but untested with dual-firewall setup (prod pfSense + PoC OPNsense sharing same gateway). Must verify before relying on it.
- Failover: **out of scope for phase 1**
- PPPoE credentials: stored securely, not in chat/docs

---

## Network Topology

```
[ISP feeds]
  Proximus → VLAN 10 → OPNsense WAN1 (PPPoE)
  Telenet  → VLAN 999 → OPNsense WAN2 (static, preferred for WG)

[OPNsense]
  WAN1/WAN2 → dual WAN (failover not configured yet)
  WireGuard → remote admin access
  LANs → PoC VLANs 300/310/320/330/340/350/400/410

[PoC internal]
  10.1.0.0/20 split by VLAN/zone

[Prod OOB]
  10.6.224.x remains separate

srv-proxmox-prod-01 (OOB 10.6.224.x)
  └── Rune VM (10.6.224.x)
        └── static route: 10.1.0.0/20 → 10.6.224.20

srv-proxmox-poc-01 (OOB 10.6.224.x)
  └── OPNsense VM
        ├── WAN: 10.6.224.20  ← Rune reaches it here
        ├── MGMT  (310): 10.1.1.1
        ├── DMZ   (320): 10.1.2.1
        └── SVC   (330): 10.1.3.1
              └── all PoC VMs
```

---

## VLAN Plan

Collision check performed against live switch configs (3com OOB, Arista 7060, Arista 7020).

| VLAN | Name | Subnet | Purpose |
|---|---|---|---|
| 300 | POC-OOB | 10.1.0.0/24 | Proxmox IPMI, device mgmt |
| 310 | POC-MGMT | 10.1.1.0/24 | Rune VM, Terraform, Ansible |
| 320 | POC-DMZ | 10.1.2.0/24 | Traefik, WireGuard |
| 330 | POC-SVC | 10.1.3.0/24 | All service VMs |
| 340 | POC-STORAGE | 10.1.4.0/24 | **TBD** — storage access (see notes below) |
| 350 | POC-BACKUP | 10.1.5.0/24 | **TBD** — may be removed (see notes below) |
| 400 | POC-WIFI | 10.1.6.0/24 | Unifi AP users |
| 410 | POC-USERS | 10.1.7.0/24 | Workstations (future) |

### Collision check — all clear ✅

| VLAN range | In use | Avoided |
|---|---|---|
| 10, 20 | Proximus ISP (3com OOB) | ✅ |
| 110, 111, 120, 121 | P2P RED/BLUE fabric (Arista 7060+7020) | ✅ |
| 600, 605, 606 | MGMT_CTRL, PTP, VRF leaking | ✅ |
| 640, 740 | SMPTE 2110 RED/BLUE broadcast fabric | ✅ |
| 999 | Telenet Fiber Corp (3com OOB) | ✅ |
| **300–410** | **Clean — nothing in this range on any switch** | ✅ |

---

## Tool Placement

| Tool | VM / Location | VLAN | Notes |
|---|---|---|---|
| OPNsense | VM on poc-01 | WAN + 310/320/330 | Core firewall/router |
| Proxmox SDN | poc-01 host | — | Virtual zones first |
| Terraform | Dedicated VM | 310 (MGMT) | Not on Rune VM — migrate state from Rune VM |
| step-ca | Dedicated VM | 330 (SVC) | Internal CA |
| Traefik | Dedicated VM | 320 (DMZ) | Reverse proxy, public + internal ingress |
| NetBox | Dedicated VM | 330 (SVC) | IPAM + documentation |
| Vaultwarden | Dedicated VM | 330 (SVC) | Password manager (Bitwarden-compatible) |
| Authentik | Dedicated VM | 330 (SVC) | Internal SSO — federates with EntraID via OIDC |
| Storage | TBD | TBD | See storage notes — NAS is on OOB, not PoC VLANs |
| Rune VM | prod-01 OOB | 10.6.224.x | Reaches PoC via static route → 10.6.224.20 |

---

## Firewall Policy

**Phase 1: not designed yet.**

Phase 2 decisions to make:
- Floating rules vs per-VLAN (prefer floating if policy is identical across VLANs)
- DNS enforcement: redirect port 53 → OPNsense Unbound
- NTP enforcement: redirect port 123 → OPNsense ntpd
- DoT: Cloudflare 1.1.1.1:853 upstream from Unbound
- Zone isolation: MGMT can reach SVC, SVC cannot initiate to MGMT

---

## Terraform State Notes

- Current state: **local backend**, test-only — not production Terraform work yet
- NAS backup workaround: `scripts/backup-state.py` → NAS `/by-terraform-state/poc/terraform.tfstate`
- NAS is on OOB (10.6.224.6) — new TF VM will need OOB access or NFS mount to reach it
- Migration: trivial — state is small, was test-only. Discuss approach when VM is ready.
- Future backend: S3-compatible (MinIO or GitLab) at phase 5 per ADR-0011
- Cleanup needed: externalize `NAS_PASS` from `backup-state.py` (currently hardcoded)

---

## Storage Notes

- NAS (10.6.224.6) is on **OOB network** — not on PoC VLANs
- NAS is effectively accessed like an external/ISP resource from PoC perspective
- VLAN 340/350 exist as placeholders but their purpose depends on:
  - Can PoC VMs route to OOB (10.6.224.x) via OPNsense?
  - Or does NAS need to be on a PoC-accessible segment?
- This needs to be **verified when OPNsense is up**
- Until then: 340/350 are reserved but not assigned
- NFS mount on TF VM as backup path is valid — discuss when VM is ready

---

## Open Decisions

| # | Question | Status |
|---|---|---|
| 1 | Storage: how do PoC VMs reach NAS on OOB? Route via OPNsense or separate NIC? | Open — verify when OPNsense up |
| 2 | VLAN 340/350: keep, repurpose, or remove? | Depends on #1 |
| 3 | Telenet dual-FW: can PoC OPNsense share same gateway as prod pfSense? | Must test |
| 4 | Vault PKI alongside step-ca? | Deferred to phase 2 |
| 5 | Floating vs per-VLAN firewall rules | Deferred to phase 2 |
| 6 | NetBox → Terraform sync model | Deferred to phase 2 |
| 7 | WAN failover design | Explicitly out of scope |
