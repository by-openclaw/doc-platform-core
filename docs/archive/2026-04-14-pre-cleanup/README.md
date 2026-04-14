# Pre-cleanup archive — 2026-04-14

This folder contains the **loose files** that lived directly under `docs/` before the scoped ADR refactor (completed 2026-04-14) absorbed their decisions into `docs/adr/{scope}/` files and `OPERATING-STANDARD.md`.

## Why these files are here

These 10 files predated the scoped ADR refactor. They were authoritative when written, but by 2026-04-14 every committed decision they contained was either:

- **Re-expressed** in a scoped ADR under `docs/adr/{scope}/`
- **Re-expressed** in `OPERATING-STANDARD.md`
- **Stale** (referenced pre-refactor state — e.g. `pfSense` instead of `OPNsense`, old VLAN IDs, old FQDN pattern with env in hostname)
- **Speculative** (Tier 2 broadcast / VoIP / CCTV modules that were never actually built)

Keeping the files in active `docs/` alongside the scoped ADRs created two risks:
1. **Contradiction** — stale content in a loose file disagreed with the current scoped ADR, and readers had no way to know which was authoritative
2. **Clutter** — 10 files of ~5000 lines that duplicated decisions the scoped ADRs now owned cleanly

Archiving them preserves the content for historical reference while removing them from the active documentation surface.

## Per-file disposition

| Archived file | Original size | What replaced it | Content loss |
|---|---|---|---|
| `architecture.md` | 538 lines | **Text**: `infra/0001-platform-stack §Cross-reference`, `infra/0004-network-architecture`, `identity/0001-authentication §Architecture`, `git/0002-platform-strategy`. **Visual diagrams**: no direct replacement — if visual aids are wanted later, regenerate from the current scoped ADRs. | **None** for text content. Visual diagrams need regeneration from scoped ADRs (existing diagrams reference `pfSense`, old VLAN IDs, old FQDN patterns — they are unsafe to reuse as-is). |
| `netbox.md` | 381 lines | `services/0003-netbox-cmdb` fully covers every decision (data model, role vocabulary, plugin pipeline, sync architecture, deployment parameters). CE product documentation (features list, Enterprise differences) is reference material, not a platform decision. | **None**. Every decision is in `services/0003-netbox-cmdb`. NetBox CE product documentation is available upstream at [netbox-community/netbox](https://github.com/netbox-community/netbox). |
| `idempotency-strategy.md` | 379 lines | Per-tool idempotency is spread across the scoped ADRs that own each tool: `infra/0003-terraform-standard`, `lib/python/0001-design-standard`, `security/0001-secret-storage`, `services/0004-database-strategy`, `infra/0006-logging`, `infra/0007-monitoring`, `security/0003-hardening`. Generic script patterns live in `OPERATING-STANDARD.md §§5.3.1–5.3.6`. | **None**. |
| `naming-convention.md` | 768 lines | Split across 4 scoped ADRs: `naming/0001-infra` (hosts, VMs, LXCs, FQDNs, DNS), `naming/0002-identity` (account and group patterns), `naming/0003-firewall` (OPNsense aliases and rules), `naming/0004-automation` (repo, Python, Ansible, Terraform, test naming). | **None**. All 4 scoped ADRs corrected stale conventions from the flat file (e.g. env is no longer in hostnames per `naming/0001-infra §10 Stability rule`). |
| `poc-platform-design.md` | 211 lines | `infra/0001-platform-stack` + `infra/0002-platform-charter §Layer 1 Proxmox Base`. | **None**. Phase 1 design is captured in the layer model. |
| `poc-platform-scope-v1.md` | 165 lines | — | **Intentional**. Self-labeled "Draft — pending Opus audit, not yet tested, commands/configs are design intent not verified". A brainstorming artifact that was always transient. Never committed as an authoritative decision. |
| `proposal-ci-platform-design.md` | 585 lines | `git/0001-workflow`, `git/0002-platform-strategy`, `git/0003-configuration`, `OPERATING-STANDARD.md §4.4 PR Content Standard`. | **None**. Self-labeled "Proposal" — preliminary, superseded by the merged git scope ADRs. |
| `2026-04-01-infra-bom-poc-proxmox.md` | 528 lines | `infra/0001-platform-stack` (tool inventory) + `security/0005-licensing-policy` (license flags). | **None**. Dated BoM superseded. |
| `stack.md` | 467 lines | `infra/0001-platform-stack §1–§4` (Compute / IaC / Storage / Docs) + `infra/0001-platform-stack §Cross-reference` (pointer to all scoped ADRs) + `security/0005-licensing-policy §4 Flagged decisions` (per-tool license decisions) + per-tool `docs/licensing.md` files under `platform-setup/tools/{tool}/`. | **Partial — detailed 29-tool inventory not duplicated**. `infra/0001-platform-stack` is a ~170-line pointer ADR, not a full catalog. The detailed inventory is preserved here in the archive if it's needed later — either restore it (with updates) to `platform-setup/docs/stack.md` in a follow-up, or regenerate from per-tool licensing.md files. **Tier 2 modules (broadcast / VoIP / CCTV) were speculative** — they remain here as historical planning but are not a committed platform decision. |
| `vm-template-spec.md` | 463 lines | `infra/0002-platform-charter §Layer 1 Proxmox Base` (the *decision* that VMs come from a cloud-init template) + the future `platform-setup/tools/proxmox/runbooks/vm-template.md` (the *spec* for building the template — to be written). | **Partial — operational spec for building the VM template not duplicated in a scoped ADR**. `infra/0002` owns the decision to use cloud-init templates; this file was the build spec. If/when a Proxmox VM template runbook is written, the relevant bits can be cherry-picked from here. Self-labeled "Draft v4" — was never production-final. |

## How to use this archive

- **Read-only.** These files are **never edited**. If a decision needs updating, update the scoped ADR that supersedes it, not this archive.
- **Historical reference.** Use when you need to understand pre-refactor state — "what did we think in 2026-04 before the refactor?" Auditors asking "show me the trail" find it here.
- **Cherry-pick source for runbooks.** If a follow-up runbook needs content from `stack.md` or `vm-template-spec.md`, copy (with updates) into the new runbook file. Do not link directly from active documentation.
- **Not authoritative.** Nothing in this folder is a current source of truth. The scoped ADRs under `docs/adr/{scope}/` and `OPERATING-STANDARD.md` are the authoritative sources.

## Related archives

- `docs/adr/archive/` — pre-refactor snapshots of the 34 **flat ADRs** that were consolidated into the 30 scoped ADRs (captured in PR #13). Same read-only rule applies.
- This folder (`docs/archive/2026-04-14-pre-cleanup/`) — pre-refactor snapshots of the 10 **loose files** under `docs/` (captured in this PR).

Both archives together form the complete pre-refactor record. Nothing from before 2026-04-14 is lost; everything is preserved as an immutable snapshot.
