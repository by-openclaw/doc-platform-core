# ADR-0027 — OPNsense Provisioning Contract

**Date:** 2026-04-04
**Status:** Accepted
**Deciders:** Youssef Boujraf
**Tags:** `opnsense`, `provisioning`, `bootstrap`, `terraform`, `ansible`

---

## Context

OPNsense is a FreeBSD-based firewall appliance. Unlike Linux VMs it cannot be provisioned via cloud-init. This creates an ambiguous boundary between what is manual, what is Terraform, and what is Ansible. Without a documented contract, configuration drifts and sessions are wasted on manual UI work that should be automated.

---

## Decision

Define a strict, phased provisioning contract with clear ownership per step.

### Phase 0 — Terraform: VM Lifecycle

Terraform owns the VM shell only:

| Item | Value |
|---|---|
| Module | `modules/vm-opnsense` |
| Output | VM created, ISO attached, NICs assigned, tags set |
| QEMU agent | `enabled = false` at this stage |
| No-touch list | OS config, firewall rules, network config, packages |

### Phase 1 — Manual Bootstrap (noVNC console, once only)

Minimum viable: get SSH working so Ansible can take over.

| Step | Action |
|---|---|
| 1 | Complete OPNsense ISO installer |
| 2 | Assign interfaces: `vtnet0` = WAN, `vtnet1` = LAN |
| 3 | Set LAN IP: `10.1.1.1/24` |
| 4 | Enable SSH: System → Administration → Secure Shell |
| 5 | Enable shell access: same page |
| 6 | Add root authorized_keys: `rune@by-systems-automation` + `by-systems@ws-win11-ref` |

**That is all.** Nothing else is done manually. No firewall rules, no gateways, no routes, no packages, no UI config.

### Phase 2 — Ansible: Full Configuration

Ansible owns everything after Phase 1.

| Task file | Responsibility |
|---|---|
| `system.yml` | Hostname, domain, password auth disabled |
| `qemu-agent.yml` | Install `os-qemu-guest-agent` |
| `interfaces.yml` | VLAN sub-interfaces: vtnet1.310 (MGMT), vtnet1.320 (DMZ), vtnet1.330 (SVC) |
| `dhcp.yml` | MGMT DHCP pool `10.1.1.100–199` |
| `dns.yml` | Unbound: DoT upstream, listening on MGMT+SVC |
| `ntp.yml` | NTP upstream + serve internal |
| `aliases.yml` | All firewall aliases (host/port/net/url) |
| `rules.yml` | All firewall rules |
| `wireguard.yml` | Plugin install + peer config (Phase 4) |

Collection: `ansibleguy.opnsense`
Inventory host: `vm-opnsense-01` (SSH via `10.6.224.106` WAN, API via `10.1.1.1` LAN)
API key: `svc-rune` — created manually once in OPNsense UI → stored in `ansible-vault`

### Phase 3 — Terraform: Post-Ansible Finalization

After Ansible confirms `os-qemu-guest-agent` is running:

| Item | Action |
|---|---|
| Agent | Flip `agent { enabled = true }` → `terraform apply` |
| ISO | Remove/detach cdrom block → `terraform apply` |
| Protection | Set `protection = true` when VM is stable |

---

## Consequences

- Zero manual UI work after Phase 1 bootstrap. All config is code.
- Any manual UI change is a drift event — must be followed immediately by updating the Ansible role.
- API key creation for `svc-rune` is the only permanent manual step (cannot be automated before API exists).
- Config backup (System → Configuration → Backups) must be committed to `tools/opnsense/config/` after each Ansible run.

---

## References

- ADR-0003: OPNsense VM configuration standard (CPU, RAM, firmware, SCSI)
- ADR-0010: Naming convention
- ADR-0012: Env tier per-VM
- ADR-0014: TLS/CA strategy
- ADR-0015: Network/VLAN architecture
- `platform-setup/tools/opnsense/docs/fact-matrix.md` — live vs documented state
- `ansible-platform/roles/opnsense/` — Ansible role
- `infra-terraform-proxmox/modules/vm-opnsense/` — Terraform module
