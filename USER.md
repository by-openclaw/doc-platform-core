# USER.md - About My Lord

> **Scope:** Workspace-level — describes the human owner across all repos and sessions
> **Related:** SOUL.md (agent identity), MEMORY.md (session memory)

- **Name:** Youssef Boujraf
- **What to call him:** Youssef
- **Handle:** yboujraf
- **Org:** BY-SYSTEMS (founder/operator)
- **Timezone:** UTC (based in Belgium)
- **Discord:** yboujraf (#bot-openclaw is his primary channel with me)

## Who He Is

Systems engineer and operator. Runs BY-SYSTEMS — a DevOps/IT/Telecom/Broadcast integrator working in CISO-grade environments. Technically sharp, high standards, low tolerance for spaghetti.

Knows exactly what good looks like. Gets frustrated when things are "done but not really done." Wants a platform he can trust — not one he has to babysit.

## What He Cares About

- **Correctness over speed.** Would rather wait and get it right than ship something half-baked.
- **No surprises.** If something changed, it should be documented, tagged, and visible.
- **Automation that actually works.** Not "it ran once." Idempotent. Tested. Dry-run. CI-gated.
- **Clean repos.** Convention-driven, no duplication, clear references, structured methodology.
- **Non-blocking workflow.** Subagents, async Discord notifications — he should be able to fire and forget.

## What Frustrates Him

- Having to go back and verify if something was actually done
- Partial updates — CLAUDE.md updated but CHANGELOG missed, or RAID skipped
- Features that "work" but have no tests, no lint, no ensure pattern
- Accumulating technical debt disguised as progress
- Generic, default, empty files (like this one was before today)

## What He's Building

BY-SYSTEMS PoC platform — full DevOps stack on Proxmox:
- Network: Arista 7060/7020, pfSense, VLAN segmentation
- Infra: Terraform (bpg/proxmox), Ansible, cloud-init
- Platform: NetBox, GitLab CE, Vault, Authentik, Teleport, k3s
- Observability: Prometheus, Grafana, Loki, Wazuh
- Storage: Synology DS1513+ (NAS), MinIO
- Agent: Rune (me) on an Ubuntu 24.04 Rune VM

## His Reference Standard

His brother uses the same tools, spends less time, and ships clean work — commits, CHANGELOG, tags, diagrams, all synced without being asked. That's the bar. We should meet it.

## Context in Discord

- In group/channel contexts: use `memory_search` / `memory_get` on demand — don't load full MEMORY.md
- `#bot-openclaw` is private (owner only) — can speak freely here

## Notes

- Prefers direct, structured, reasoning-forward responses — no fluff
- Will call out vagueness immediately — don't hedge, be specific
- Appreciates when I push back if something is wrong or incomplete
- Hates wasted time more than being told something is harder than expected
- When pushing back: lead with the fact or risk, then the recommendation. No preamble, no softening. "This will break X because Y. Recommend Z instead."

## Update Policy
- Updated when learning new preferences, frustrations, or context
- Never updated with ephemeral session details (those go in MEMORY.md)
- Changes communicated in the session they are made
