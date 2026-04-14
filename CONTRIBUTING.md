# Contributing to `doc-platform-core`

> Read [`OPERATING-STANDARD.md`](OPERATING-STANDARD.md), [`README.md`](README.md), and [`CLAUDE.md`](CLAUDE.md) before starting.
> **This is a docs-only repo.** No code, no scripts, no CI pipelines.

---

## Before you start

1. [`OPERATING-STANDARD.md`](OPERATING-STANDARD.md) — mandatory platform rules (applies to every BY-SYSTEMS repo and every session)
2. [`docs/adr/README.md`](docs/adr/README.md) — scoped ADR index (30 ADRs across 7 scopes)
3. [`docs/adr/naming/0001-infra.md`](docs/adr/naming/0001-infra.md) — **mandatory before creating any infrastructure-related file**
4. [`docs/adr/naming/0004-automation.md`](docs/adr/naming/0004-automation.md) — **mandatory before creating any repo, code, or test artifact**
5. [`docs/raid.md`](docs/raid.md) — check open risks and issues before starting work

---

## Branch naming

Per `OPERATING-STANDARD.md §4.3`:

```text
{type}/{short-description}-{issue-id}
```

Examples:
- `docs/add-networking-section-42`
- `chore/rename-templates-15`
- `fix/broken-adr-links-7`

---

## Commit types — docs repo only

This repo uses a **limited set** of Conventional Commits types. Code-oriented types are rejected.

| Type | Use when |
|---|---|
| `docs` | Documentation content changes — new sections, rewrites, clarifications |
| `chore` | Maintenance — renames, reorganisation, template updates, tidying |
| `fix` | Corrections — broken links, outdated references, typos, wrong paths |

**Do not use** `feat`, `ci`, `build`, `refactor`, `test` in this repo. The CI guard rejects PRs that use code-oriented commit types here.

---

## ADR process

### Where ADRs live

All platform ADRs are **scoped** under `docs/adr/{scope}/`:

```text
docs/adr/{scope}/NNNN-short-title.md
```

Current scopes (see [`docs/adr/README.md`](docs/adr/README.md) for the full table):

- `identity/` — authentication, provisioning, machine credentials, OS accounts
- `git/` — workflow, platform strategy, configuration
- `naming/` — infra, identity, firewall, automation naming patterns
- `security/` — secret storage, compliance, hardening, certificates, licensing
- `infra/` — stack, charter, terraform, network, env tiers, logging, monitoring, backup
- `services/` — opnsense, email, netbox CMDB, database, notifications
- `lib/python/` — Python library design standard

### When to create a new ADR

Create a new ADR when you need to **record an architectural decision** about:

- Platform architecture, topology, or layer model
- Tool selection, replacement, or removal
- Security, identity, secret storage, or certificate strategy
- Naming, environment tier, or DNS conventions
- A new service's deployment contract or data model
- A new library design standard

### How to add an ADR

1. Find the correct scope folder under `docs/adr/{scope}/`
2. Check existing ADRs to determine the next sequential number (**per-scope numbering**, not global)
3. Use the format `docs/adr/{scope}/NNNN-short-title.md` (lowercase, hyphens, zero-padded)
4. Every ADR must include:
   - Status (`Draft` / `Accepted` / `Deprecated` / `Superseded by ...`)
   - Date (ISO 8601)
   - Scope (one-line description of what this ADR owns)
   - Related ADRs (cross-references to peer ADRs in other scopes)
   - Context, Decision, Consequences
   - **`CISO mapping` section** — mandatory per `docs/adr/security/0002-compliance-mapping §ADR authoring rule`. If no direct control mapping, write `N/A — no direct control mapping`.

### ADR cross-references

Reference other ADRs by their **scoped path**, not by flat number:

- ✅ `naming/0001-infra §7 Env sub-domain`
- ✅ `security/0001-secret-storage §KV path convention`
- ❌ `ADR-0010`, `ADR-0001`, `ADR-0029` — flat ADRs are archived, never reference them

---

## File naming

- All lowercase, hyphens, `.md` extension
- Dated docs (when appropriate): `{type}-{subject}-{YYYY-MM-DD}.md`
- Templates: `{name}.tpl.md`

Examples:
- `docs/templates/README.tpl.md`
- `docs/archive/2026-04-14-pre-cleanup/stack.md`

---

## Definition of Done — PR checklist

Per `OPERATING-STANDARD.md §5.1`:

- [ ] File naming follows `docs/adr/naming/0004-automation.md §8`
- [ ] All paths reference current scoped ADR structure (no flat ADR numbers, no archived file paths)
- [ ] Every new ADR has a `CISO mapping` section (or explicit `N/A`)
- [ ] Markdown renders correctly (no broken tables, links, or images)
- [ ] Cross-references to other ADRs use scoped paths (`{scope}/NNNN-title`)
- [ ] [`docs/raid.md`](docs/raid.md) updated if a new risk / issue / dependency is discovered
- [ ] PR description follows `OPERATING-STANDARD.md §4.4 Pull Request Content Standard`
- [ ] `CHANGELOG.md` is auto-maintained by Release Please — **never edit manually**

---

## Archive policy

Historical docs move to the appropriate archive folder — **never delete**:

| Archive folder | Contents |
|---|---|
| [`docs/adr/archive/`](docs/adr/archive/) | Pre-refactor snapshots of the 34 flat ADRs that were consolidated into the 30 scoped ADRs |
| [`docs/archive/`](docs/archive/) | Pre-cleanup snapshots of loose files (e.g. `docs/archive/2026-04-14-pre-cleanup/`) |

**Archives are never edited.** They exist for historical reference, auditor trail, and cherry-pick during runbook authoring. Read-only.

---

## Templates

Standardized templates for new repos live in [`docs/templates/`](docs/templates/):

| Template | Purpose |
|---|---|
| `CLAUDE.tpl.md` | Per-repo Claude Code contract |
| `AGENTS.tpl.md` | Per-repo generic agent onboarding |
| `CONTRIBUTING.tpl.md` | Per-repo contributing guidelines |
| `README.tpl.md` | Per-repo README |

Update templates when a repo-level improvement is proven — backport pattern: improve one repo, promote to template, propagate to other repos.

---

## Release process

- Use Conventional Commits consistently (see §Commit types)
- [Release Please](https://github.com/googleapis/release-please) determines version bumps automatically
- **Never manually edit `CHANGELOG.md` or version strings**
- PRs are merged with `gh pr merge N --merge --delete-branch` (merge commit) per `docs/adr/git/0001-workflow.md`
- `@yboujraf` is sole merge authority — agents open PRs, never merge

---

## Agent boundary

Per `docs/adr/git/0001-workflow.md §Agent boundary`:

- Agents **may** commit to feature branches, open PRs, self-review for CI/lint
- Agents **may not** push directly to `main`, merge PRs, or force-push without explicit per-action user approval
