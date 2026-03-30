# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/).

---

## [0.5.1](https://github.com/by-openclaw/doc-platform-core/compare/v0.5.0...v0.5.1) (2026-03-30)


### Bug Fixes

* correct pull_request trigger in project-board-sync workflow ([f94e64d](https://github.com/by-openclaw/doc-platform-core/commit/f94e64de4056f4ec0fa4ae08049cf0a0958ae29b))

## [Unreleased]

## [0.5.0] — 2026-03-29

### Added
- `docs/adr/0008-terraform-state-management.md`: Terraform state on Synology NAS, restore procedure, FileStation auth quirk, Phase 2 GitLab migration path

### Fixed
- `docs/adr/0006-platform-charter.md`: OOB gateway corrected `10.6.255.254` → `10.6.224.1`

## [0.4.0] — 2026-03-28

### Added
- `docs/adr/0006-platform-charter.md`: Platform charter — layer model (0–5), asset structure, network topology, VM baseline
- `docs/adr/0007-automation-scripting-standard.md`: Scripting and automation standards — separation of concerns, Ansible/Terraform/Python patterns
- `docs/raid.md`: RAID log — Risks, Assumptions, Issues, Dependencies

## [0.3.0] - 2026-03-26

### Changed
- Anonymized all org/customer references for public repo — `BY-SYSTEMS` → `ORG`, domain references sanitized, customer names replaced with generic identifiers

## [0.2.0] - 2026-03-26

### Added
- `docs/netbox.md` — NetBox deep-dive: CMDB/IPAM/DCIM source of truth, object model, platform integrations, idempotent usage strategies, naming conventions for devices/prefixes/VLANs/racks/tenants/sites
- `docs/idempotency-strategy.md` — idempotency patterns per tool (Terraform, Ansible, GitLab CI, Vault, Prometheus, etc.), GitOps principles, free plan features and limitations for all stack components
- `docs/archive/` — archive directory for superseded documents

### Changed
- `CLAUDE.md` — updated repo layout to include new files and archive directory
- `README.md` — updated contents table with new documentation files
- `docs/roadmap.md` — updated brainstorm reference to archived path
- `docs/adr/0001-platform-stack-decisions.md` — updated brainstorm reference to archived path

### Moved
- `docs/brainstorm-2026-03-25.md` → `docs/archive/brainstorm-2026-03-25.md` — session notes archived; all decisions formalized in `stack.md`, `naming-convention.md`, and `adr/0001`

## [0.1.0] - 2026-03-26

### Added
- `docs/stack.md` — full technology stack with tools, roles, rationale
- `docs/naming-convention.md` — naming rules for repos, branches, FQDNs, devices, SSH/GPG keys, assets, containers, K8S, mail, LFS
- `docs/architecture.md` — platform architecture in PlantUML (5 diagrams)
- `docs/roadmap.md` — phased delivery plan (5 phases)
- `docs/raid.md` — risks, assumptions, issues, dependencies
- `docs/adr/0001` — stack decisions ADR
- `docs/templates/` — SOW, README, CONTRIBUTING, CLAUDE templates
- `assets/diagrams/` — PlantUML source files
- `CLAUDE.md` — agent instructions and repo conventions
- `.gitattributes` — Git LFS config for binary assets
- `.gitignore` — standard exclusions
- `README.md` — project overview

---

_Scope: doc | Status: Draft, PoC phase_

[Unreleased]: https://github.com/by-openclaw/doc-platform-core/compare/v0.5.0...HEAD
[0.5.0]: https://github.com/by-openclaw/doc-platform-core/compare/v0.4.0...v0.5.0
[0.4.0]: https://github.com/by-openclaw/doc-platform-core/compare/v0.3.0...v0.4.0
[0.3.0]: https://github.com/by-openclaw/doc-platform-core/compare/v0.2.0...v0.3.0
[0.2.0]: https://github.com/by-openclaw/doc-platform-core/compare/v0.1.0...v0.2.0
[0.1.0]: https://github.com/by-openclaw/doc-platform-core/releases/tag/v0.1.0
