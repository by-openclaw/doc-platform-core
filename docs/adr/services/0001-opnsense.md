# services/0001 — OPNsense

**Status:** Draft
**Date:** 2026-04-14 (supersedes flat ADR-0027, 2026-04-04)
**Scope:** Ownership contract between Terraform, manual bootstrap, and Ansible for OPNsense firewall deployment. Does not define VLAN assignments, firewall rules, WireGuard peer naming, or step-by-step bootstrap procedures — each has its own authoritative location.
**Related:** `infra/0002-platform-charter`, `infra/0004-network-architecture`, `naming/0003-firewall`, `identity/0004-os-accounts`, `security/0001-secret-storage`, `security/0004-certificate-strategy`

---

> **This ADR is an exception, not a pattern.** Every other service on the platform (GitLab, NetBox, Authentik, Vault, Mailcow, Grafana, Prometheus, Loki, etc.) follows the **generic Linux VM pipeline** defined by `infra/0002-platform-charter §Layer 1` → cloud-init → `identity/0004-os-accounts §9` → `security/0003-hardening` → service-specific Ansible role. Those services do **not** need their own provisioning contract ADR.
>
> OPNsense is documented separately because it breaks the generic pipeline in six concrete ways — see §Context below.

## Context

OPNsense is a FreeBSD-based firewall appliance. Unlike Linux VMs, it **cannot** be provisioned via cloud-init — the ISO installer is interactive, and the API is unreachable until SSH is manually enabled on first boot. This creates an ambiguous boundary between what is manual, what is Terraform, and what is Ansible. Without a documented contract, configuration drifts and sessions are wasted on manual UI work that should be automated.

### Six constraints that break the generic Linux VM pipeline

| Constraint | Why the generic pipeline can't cover it |
|---|---|
| FreeBSD-based, ships without cloud-init | Phase 1 must be manual (ISO installer) |
| API is unreachable until SSH is manually enabled | Cannot Ansible-in via a pre-provisioned template |
| UI shell of `None` kills SSH silently | OPNsense-specific quirk not in the generic `user-mgmt` role |
| Manual API key generation | One permanent manual step, cannot be automated before API exists |
| `pfctl -d` required during bootstrap | No equivalent for Linux VMs |
| No `qemu-guest-agent` by default | Requires two-phase Terraform apply (pre-Ansible / post-Ansible) |

Each constraint individually might be worked around; together they force a dedicated provisioning model.

## Decision

Provisioning is split into **four phases** with strict ownership per phase. Each phase has one owner and a narrow scope. No phase overlaps with another.

### Phase 0 — Terraform: VM lifecycle (automated)

Terraform owns the **VM shell only** — never the OS or firewall configuration inside it.

| Owns | Does not own |
|---|---|
| VM creation, CPU / RAM / disk sizing | OS configuration |
| ISO attachment (install media) | Firewall rules |
| NIC assignment to Proxmox bridges / SDN VNets | Network interface IPs |
| Proxmox tags (env, role) | Packages, services |
| Initial boot order | API keys, users |

At the end of Phase 0, OPNsense is a booted VM on the ISO installer screen. Nothing is configured inside.

**Module:** `infra-terraform-proxmox/modules/vm-opnsense`
**QEMU guest agent:** `enabled = false` at this stage (OPNsense ships without it; it is installed in Phase 2)

### Phase 1 — Manual: minimum viable bootstrap (once, console only)

Phase 1 is the **only** manual step. Its single goal: enable SSH and API access so Ansible can take over. **Nothing else.**

**Ownership boundary:**
- ✅ In scope: ISO installer, network interface assignment, LAN IP assignment, enable SSH, create `svc-rune` user, generate API key
- ❌ Out of scope: firewall rules, VLAN sub-interfaces, DHCP pools, DNS, NTP, packages, UI theming, anything else

Step-by-step procedure lives in the runbook: **`platform-setup/tools/opnsense/runbooks/bootstrap.md`**. This ADR does not reproduce the steps — runbooks drift when steps change, ADRs should not.

**Key decisions locked here:**

- **User `svc-rune` is created during Phase 1** — `admins` group, `shell=/bin/sh` (mandatory for OPNsense SSH, confirmed 2026-04-10), SSH public key per `identity/0004-os-accounts §2`
- **No root SSH needed.** `svc-rune` with `shell=/bin/sh` + public key is sufficient for Ansible. The OPNsense UI "None" shell kills SSH silently.
- **API key is generated manually in the UI**, once. There is no way to automate this before the API itself exists. The key is stored in HashiCorp Vault at `secret/{env}/opnsense/svc-rune-api-key` per `security/0001-secret-storage`.
- **LAN IP is the MGMT gateway.** Specific address per environment lives in `infra/0004-network-architecture §3` (prod) and §4 (test) — this ADR does not repeat them.
- **Firewall is temporarily disabled** (`pfctl -d` from console) during Phase 1 bootstrap to allow initial API + SSH access. Phase 2 re-enables it with managed rules.

### Phase 2 — Ansible: full configuration (automated, repeatable)

After Phase 1, Ansible owns **everything** — system settings, interfaces, VLANs, DHCP, DNS, NTP, firewall rules, WireGuard, packages.

**Collection:** `ansibleguy.opnsense`
**Role:** `ansible-platform/roles/opnsense/`
**Inventory host:** `vm-opnsense-01` (or whatever the deployment calls it per `naming/0001-infra §5`)
**Auth:** API key from Vault (`secret/{env}/opnsense/svc-rune-api-key`)

Task breakdown:

| Task file | Responsibility | Authoritative source |
|---|---|---|
| `system.yml` | Hostname, domain, password auth disabled, admin UI TLS | `security/0004-certificate-strategy` |
| `qemu-agent.yml` | Install `os-qemu-guest-agent` (enables Proxmox agent at OS level) | — |
| `interfaces.yml` | VLAN sub-interfaces on the trunk NIC | `infra/0004-network-architecture §3` / §4 |
| `dhcp.yml` | DHCP pools per VLAN | `infra/0004-network-architecture` |
| `dns.yml` | Unbound resolver, DoT upstream | `infra/0006-logging` (for query logging), `naming/0001-infra §9` (for zone structure) |
| `ntp.yml` | NTP upstream + serve internal | — |
| `aliases.yml` | Firewall aliases | `naming/0003-firewall` |
| `rules.yml` | Firewall rules | `naming/0003-firewall` |
| `wireguard.yml` | Plugin install, server key, peer config | `infra/0004-network-architecture §8`, `security/0001-secret-storage` |

**Rules:**
- Every task file reads its data from a cross-referenced ADR or NetBox — no standalone truth in Ansible vars
- VLAN IDs, subnets, and zone names are **never hardcoded** in the playbook; they come from `infra/0004-network-architecture` via Ansible variables or NetBox dynamic inventory
- Firewall alias and rule names follow `naming/0003-firewall` (no `alias_` prefix, `{ACTION} {source}→{destination} {service}` description format)
- WireGuard server private key is **generated once by Ansible**, stored in Vault immediately, never exported to disk again

### Phase 3 — Terraform: post-Ansible finalization

After Ansible confirms `os-qemu-guest-agent` is running inside the VM, Terraform flips two flags:

| Item | Action |
|---|---|
| QEMU guest agent | `agent { enabled = true }` → `terraform apply` |
| Install media | Remove / detach CDROM block → `terraform apply` |
| VM protection flag | `protection = true` when the VM is stable in production |

These are Terraform-owned because they change the VM shell, not the OS. They cannot run in Phase 0 because the agent has to be installed inside first.

## Drift policy

**Any manual UI change after Phase 1 is a drift event.** It must be followed **immediately** by updating the Ansible role with the same change, re-running Ansible in check mode to verify no diff, then re-running in apply mode to confirm idempotence. Drift events are tracked as incidents, not as "small tweaks".

OPNsense configuration backup (System → Configuration → Backups) is committed to `platform-setup/tools/opnsense/config/` after every successful Ansible run, as the last-resort recovery artifact.

## Consequences

- **Zero manual UI work after Phase 1.** All OPNsense configuration is code — aliases, rules, VLANs, DHCP, DNS, NTP, WireGuard.
- **The manual Phase 1 is narrow and one-off.** 8 steps, runbook-documented, never repeated for the same VM.
- **API key creation is the only permanent manual step** — it cannot be automated before the API exists. Mitigated by storing the key in Vault immediately and rotating per `security/0001-secret-storage` rotation policy.
- **Drift is an incident**, not a convenience. Every manual UI change triggers an Ansible role update within the same working session.
- **No VLAN tables in this ADR.** Network topology lives in `infra/0004-network-architecture`. This ADR references it — if you need VLAN numbers, go there.
- **Cross-OS note:** OPNsense is FreeBSD and cannot participate in the Linux SSH / Windows WinRM split from `git/0003-configuration §11`. Ansible uses SSH to reach OPNsense, always.

## Revision triggers

Revise this ADR when:
- OPNsense ships a cloud-init compatible variant (would eliminate Phase 1 entirely)
- The provisioning boundary changes (e.g. Ansible takes over Phase 0 VM creation via Proxmox modules)
- `ansibleguy.opnsense` is replaced by a different collection (e.g. `community.opnsense`)
- A second OPNsense instance is deployed for HA (CARP / pfsync) — adds multi-node coordination to Phase 2
- OPNsense API authentication changes (e.g. OIDC-based, once Authentik is live)
- The API key manual step is replaced by a programmatic enrollment flow

## CISO mapping

| Framework | Controls covered |
|---|---|
| ISO 27001:2022 | A.8.9 (configuration management — all config as code after Phase 1), A.8.32 (change management — drift is an incident, not a shortcut), A.8.20 (networks security — OPNsense firewall fully managed) |
| NIS2 | Art. 21(2)(e) (network and information systems security — automated, reproducible firewall configuration) |
