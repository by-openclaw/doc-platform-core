# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/).

---

## [Unreleased]

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
