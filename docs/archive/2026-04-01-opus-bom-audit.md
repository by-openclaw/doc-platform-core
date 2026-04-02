# Opus Audit — BoM PoC Proxmox Infra

> **From:** Opus (auditor)
> **To:** @Rune + @yboujraf
> **Date:** 2026-04-01
> **Ref:** [BoM on GitHub](https://github.com/by-openclaw/doc-platform-core/blob/main/docs/2026-04-01-infra-bom-poc-proxmox.md)
> **Verdict:** ✅ APPROVED with 3 findings (1 critical, 2 medium)

---

## Overall Assessment

Excellent work. The BoM is comprehensive, well-structured, follows the ADR-0006 layer model, and correctly identifies unknowns (VLAN 340/350 TBD, Telenet untested). Resource sizing is conservative and appropriate for PoC. The phased WAN migration is smart — safety-first with OOB fallback always available.

---

## Answers to Rune's 5 Questions

### Q1: VLAN 340/350 — NAS on OOB, not on PoC VLANs

**Finding: MEDIUM — isolation breach if not handled**

The BoM header says "fully isolated from prod (zero OOB dependency)" but the storage strategy routes NFS to `10.6.224.6` (OOB network). This contradicts the isolation claim.

**Options (for @yboujraf to decide):**
- **(A) Route via OPNsense** — add a static route + strict firewall rules (only NFS 2049/TCP from 10.1.3.0/24 → 10.6.224.6). Pragmatic for PoC. Breaks "zero OOB dependency" but limited blast radius.
- **(B) Add second NIC to NAS** — physical NIC on PoC VLAN 340. True isolation. Requires NAS config change + physical cable.
- **(C) Don't use NAS for PoC** — all persistent data on Contabo S3 + local VM disk. Cleanest isolation but limits storage options.

**Recommendation:** Option A for PoC, document the exception. Option B for prod.

VLAN 340/350 should stay **reserved** until the routing path is verified with OPNsense up. Correct call from Rune.

---

### Q2: Telenet WAN — dual-FW sharing same gateway

**Finding: LOW — correctly flagged as untested**

Two firewalls on the same L2 segment sharing a gateway works IF:
- Each has its own IP on the Telenet subnet (Telenet typically provides a /29 or /30 — verify how many usable IPs)
- No ARP conflicts — both firewalls must NOT claim the same gateway MAC
- No asymmetric routing — return traffic must go back through the correct firewall

**Risk:** If Telenet provides only 1 usable IP (common on residential/small business), this won't work. pfSense would need to release the Telenet WAN IP for OPNsense to claim it.

**Recommendation:** Verify Telenet subnet size before Phase 2. If only 1 IP: either migrate pfSense to OPNsense entirely, or use Proximus for PoC WAN (DHCP + DDNS — less stable but works). The phased approach already handles this — Phase 1 uses Proximus. Good.

---

### Q3: Authentik + EntraID — identity flow

**Finding: CORRECT — fits layer model**

The flow `service → Authentik (OIDC RP) → EntraID (OIDC OP) → Authentik → service` is standard federation. This correctly places:
- **EntraID** = upstream IdP (Microsoft tenant, external)
- **Authentik** = local broker/SP (Layer 1 — Identity & Secrets)
- **Services** = OIDC clients registered in Authentik

This fits ADR-0006 Layer 1. No separate ADR needed for EntraID itself — it's a configuration of Authentik, not a separate platform component. **However:** document the EntraID tenant ID and OIDC endpoints in the Authentik deployment runbook. For NIS2 compliance, the identity federation chain must be auditable.

---

### Q4: step-ca deferred — gaps with LE via DNS-01

**Finding: ACCEPTABLE for PoC — flag for prod**

LE + Cloudflare DNS-01 covers:
- ✅ Browser-facing TLS (`*.by-systems.be`)
- ✅ Automatic renewal via Traefik
- ✅ Works for internal services (no public HTTP needed)

Gaps:
- ❌ **No mTLS between services** — service-to-service traffic inside the network is plain HTTP. For PoC with isolated VLANs behind OPNsense firewall rules, this is acceptable. For prod, step-ca provides internal CA for mTLS.
- ❌ **No client certificates** — can't do cert-based auth for machine identity. Not needed for PoC.

**Recommendation:** Proceed without step-ca. Add it to prod roadmap when mTLS becomes a requirement (e.g., zero-trust between services).

---

### Q5: General — missing, oversized, undersized, ordered wrong

#### CRITICAL FINDING: Docker network topology across VMs

The BoM states:
> "one `traefik-net` bridge network shared across all services + Traefik"

**This cannot work.** Docker bridge networks are **per-host**. Traefik runs on `vm-traefik-poc-01` (10.1.2.10, DMZ). Services run on different VMs in SVC zone (10.1.3.x). A Docker bridge on the Traefik VM cannot reach containers on other VMs.

**How Traefik reaches services across VMs:**
- Each service VM runs Docker with its service exposed on a known port (e.g., `127.0.0.1:8080` mapped to host `10.1.3.x:8080`)
- Traefik uses **file provider** (not Docker provider) to define backends: `url: "http://10.1.3.11:8200"` for Vault, `url: "http://10.1.3.12:9000"` for Authentik, etc.
- Traefik routes based on Host header (`vault.by-systems.be` → `10.1.3.11:8200`)
- OPNsense firewall rules allow DMZ → SVC traffic on specific ports only

**Action required:** Replace "one `traefik-net` bridge network" with "Traefik file provider with static backend definitions per service VM IP". Each VM has its own local Docker bridge. Cross-VM traffic goes through the network layer (Proxmox SDN), not Docker networking.

**Alternative (future):** Docker Swarm overlay network spans hosts — but that's Phase 2, not PoC.

#### Other findings (minor)

| Item | Finding | Severity |
|---|---|---|
| Deployment order | OPNsense listed as #0 at the bottom — confusing. Should be clearly first in the list. | Low (formatting) |
| Vaultwarden IP | 10.1.3.13 — fits SVC zone. OK. | ✅ |
| GitLab 4 GB RAM | Minimum for GitLab CE. Correct. May need 6-8 GB under load — monitor. | ✅ |
| Nextcloud nginx sidecar | Correctly documented as internal fpm requirement, not a rule violation. | ✅ |
| Pi-hole on MGMT zone | Correct — DNS should be on MGMT, not DMZ. All VMs resolve via MGMT. | ✅ |
| WireGuard on OPNsense | Good call — no separate VM needed. | ✅ |
| Pre-deployment checklist | Complete and actionable. | ✅ |
| Contabo S3 buckets | 5 buckets, versioning on gitlab+backups. Correct. | ✅ |
| VLAN 1 policy | Correct — dead VLAN, native 999 blackhole. | ✅ |
| Arista trunk pre-reservation | Smart — IDs reserved now, wired later. | ✅ |

---

## Summary for @yboujraf

| # | Finding | Severity | Decision needed |
|---|---|---|---|
| 1 | Docker `traefik-net` cannot span VMs — use Traefik file provider instead | **CRITICAL** | @Rune to fix BoM |
| 2 | NAS on OOB breaks "zero isolation" claim — choose routing option A/B/C | **MEDIUM** | @yboujraf decides |
| 3 | Telenet dual-FW — verify subnet size before Phase 2 | **MEDIUM** | Verify when OPNsense is up |

**Verdict: ✅ APPROVED — proceed with deployment after fixing finding #1.**

Finding #2 and #3 can be resolved during deployment (OPNsense phase). They don't block VM creation.

---

*Opus reviewed. @Rune fixes finding #1. @yboujraf approves. Then Terraform.*
