# `doc-platform-core`

> **Scope:** `doc` | **Component:** `platform` | **Status:** `Draft — PoC phase`
> **Org:** `by-openclaw/doc-platform-core` | **License:** `CC BY-SA 4.0`

Platform documentation for the ORG internal DevOps PoC — stack decisions, architecture, naming conventions, ADRs, roadmap, and RAID log.

---

## Contents

| File | Description |
|---|---|
| `docs/stack.md` | Full technology stack — tools, roles, rationale |
| `docs/naming-convention.md` | Naming rules for repos, branches, FQDNs, devices, assets, keys |
| `docs/architecture.md` | Platform architecture diagrams (PlantUML) |
| `docs/netbox.md` | NetBox deep-dive — CMDB/IPAM/DCIM source of truth |
| `docs/idempotency-strategy.md` | Idempotency patterns, GitOps principles, free plan notes |
| `docs/roadmap.md` | Phased delivery plan |
| `docs/raid.md` | Risks, Assumptions, Issues, Dependencies |
| `docs/adr/0001-platform-stack-decisions.md` | Architecture Decision Record — stack choices |
| `docs/templates/` | Document templates (`.tpl.md`) |
| `docs/archive/` | Archived session notes and superseded docs |
| `assets/diagrams/` | PlantUML source files |
| `assets/exports/` | Rendered diagram exports (PNG) |

---

## Conventions

- All docs: Markdown, English only
- Diagrams: PlantUML source in `assets/diagrams/`, rendered via KROKI
- Binary assets tracked via Git LFS (`.gitattributes`)
- Commits follow [Conventional Commits](https://www.conventionalcommits.org/)
- Versioning follows [Semantic Versioning](https://semver.org/)

## Related

- Stack: `docs/stack.md`
- Naming: `docs/naming-convention.md`
- Decisions: `docs/adr/`
