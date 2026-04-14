# AGENTS.md — `doc-platform-core`

> **Mandatory:** [`OPERATING-STANDARD.md`](OPERATING-STANDARD.md) applies to all agents, all sessions.
> **Agent identity:** Rune. **Owner:** @yboujraf. Agent identity and owner profile live in `~/.openclaw/workspace/SOUL.md` + `USER.md` per `OPERATING-STANDARD.md §3.2 File ownership`, not in this repo.

Central documentation repository for the BY-SYSTEMS platform — 30 scoped Architecture Decision Records, operating rules, platform-wide state, and document templates. **No code lives here.**

---

## Always read first

Before creating or editing anything in this repo, read in this order:

1. [`OPERATING-STANDARD.md`](OPERATING-STANDARD.md) — platform rules, session protocol, commit discipline, quality gates
2. [`README.md`](README.md) — repo structure and overview
3. [`CLAUDE.md`](CLAUDE.md) — agent-specific constraints for this repo
4. [`docs/adr/README.md`](docs/adr/README.md) — scoped ADR index (30 ADRs, 7 scopes)
5. [`docs/adr/naming/0001-infra.md`](docs/adr/naming/0001-infra.md) — **mandatory before creating any infrastructure-related file**
6. [`docs/adr/naming/0004-automation.md`](docs/adr/naming/0004-automation.md) — **mandatory before creating any repo, code, or test artifact**
7. [`docs/raid.md`](docs/raid.md) — platform-wide risks, assumptions, issues, dependencies

---

## Commit & branch standards

- **Commit types — docs repo uses a limited set only:**
  - `docs(scope): description` — content changes
  - `chore(scope): description` — maintenance, rename, reorganise
  - `fix(scope): description` — correct errors, broken links, outdated info
  - **Do NOT use** `feat`, `ci`, `build`, `refactor`, `test` in this repo
- **Branch naming:** `{type}/{short-description}-{issue-id}` per `OPERATING-STANDARD.md §4.3`
  - Example: `docs/add-networking-adr-42`, `chore/archive-stale-docs-37`
- **ADR path convention:** `docs/adr/{scope}/NNNN-short-title.md` — **scoped folders, per-scope numbering** (not flat at `docs/adr/NNNN-*.md`)
  - Example: `docs/adr/naming/0005-foo.md`, `docs/adr/services/0006-bar.md`
  - See [`docs/adr/README.md`](docs/adr/README.md) for the current scope list
- **File names:** all lowercase, hyphens only, `.md` extension

---

## What NOT to do

> Also read [`CLAUDE.md`](CLAUDE.md) for repo layout, docs-only constraints, and key file references.

- Do **NOT** add source code, scripts, Dockerfiles, or CI pipelines (the only CI workflows here are `release-please.yml` and `project-board-sync.yml` — changes to either require @yboujraf approval per `OPERATING-STANDARD.md §5.5`)
- Do **NOT** create an ADR without first checking `docs/adr/README.md` for the correct scope folder and next sequential number
- Do **NOT** reference flat ADR numbers (`ADR-0001`, `ADR-0010`, etc.) — they are archived. Always reference the scoped path.
- Do **NOT** use `feat`, `ci`, or code-oriented commit types in this repo
- Do **NOT** add binary assets without Git LFS configured (`.gitattributes`)
- Do **NOT** write in future tense — use present tense in all documentation
- Do **NOT** add files directly at `docs/` unless they're one of the three active loose files (`status.md`, `roadmap.md`, `raid.md`). Historical content goes in `docs/archive/`, templates go in `docs/templates/`.

---

## Network naming rule — locked

Per [`docs/adr/infra/0004-network-architecture.md`](docs/adr/infra/0004-network-architecture.md) + [`docs/adr/infra/0005-environment-tiers.md`](docs/adr/infra/0005-environment-tiers.md):

- Proxmox **SDN zones** = env tier group: **`prod`** (serves `prod` + `drp`), **`test`** (serves `dev` / `test` / `staging` / `acc`)
- Proxmox **SDN VNet names are environment-agnostic**: `mgmt`, `dmz`, `svc`, `vpn`, `iot`, `voip`, `storage`, `media`, `cctv` (prod zone) / `tmgmt`, `tdmz`, `tsvc`, etc. (test zone)
- Environment context belongs in the **NetBox `env` custom field** and the **DNS zone**, not in VNet names or hostnames
- Do **NOT** invent env-prefixed VNets such as `vnet-prod-svc`, `pocmgmt`, or `vmbrPOC`
- **`poc` is NOT an env tier.** It was dropped during the scoped ADR refactor because it conflated with hostname labels. The 6 valid env tiers are `dev / test / staging / acc / prod / drp`. Any reference to `poc` outside this sentence is a bug.

---

## GitHub repo

https://github.com/by-openclaw/doc-platform-core

---

## Agent: Rune

Maintained by Rune (DevOps familiar) for the BY-SYSTEMS platform.
Owner: @yboujraf

Rune's identity and Youssef's profile live in the workspace (`~/.openclaw/workspace/SOUL.md` + `USER.md`), not in this repo. See `OPERATING-STANDARD.md §3.2`.

---

## Project stats

> Auto-updated on every release.

| Metric | Value |
|---|---|
| Scoped ADRs | 30 |
| ADR scopes | 7 (identity, git, naming, security, infra, services, lib/python) |
| Active `docs/` loose files | 3 (`status.md`, `roadmap.md`, `raid.md`) |
| Archived pre-refactor ADR snapshots | 34 (in `docs/adr/archive/`) |
| Archived pre-cleanup loose files | 10 (in `docs/archive/2026-04-14-pre-cleanup/`) |
| CI workflows | 2 (`release-please.yml`, `project-board-sync.yml`) |
