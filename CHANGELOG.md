# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/).

---

## [0.7.0](https://github.com/by-openclaw/doc-platform-core/compare/v0.6.0...v0.7.0) (2026-06-13)


### Features

* centralize agent contract files — single source of truth ([7349b86](https://github.com/by-openclaw/doc-platform-core/commit/7349b866dfdd5a2a7687d3e54d58d4a8ee04836c))


### Bug Fixes

* **agents:** link to doc-platform-core for agent contract files ([ce0dd9e](https://github.com/by-openclaw/doc-platform-core/commit/ce0dd9e81524c9385d8f50900c6f43f05f6c7a25))
* **agents:** remove unreachable OPERATING-STANDARD.md link ([87fd763](https://github.com/by-openclaw/doc-platform-core/commit/87fd76318faca453b3f6b5737466c21c0c07ebab))
* **agents:** restore OPERATING-STANDARD reference as plain text ([f0dd35e](https://github.com/by-openclaw/doc-platform-core/commit/f0dd35eb8af100d8c4926a7d9e3ef98ccdbd8e4e))

## [0.7.0](https://github.com/by-openclaw/doc-platform-core/compare/v0.6.0...v0.7.0) (2026-06-04)


### Features

* centralize agent contract files — single source of truth ([7349b86](https://github.com/by-openclaw/doc-platform-core/commit/7349b866dfdd5a2a7687d3e54d58d4a8ee04836c))


### Bug Fixes

* **agents:** link to doc-platform-core for agent contract files ([ce0dd9e](https://github.com/by-openclaw/doc-platform-core/commit/ce0dd9e81524c9385d8f50900c6f43f05f6c7a25))
* **agents:** remove unreachable OPERATING-STANDARD.md link ([87fd763](https://github.com/by-openclaw/doc-platform-core/commit/87fd76318faca453b3f6b5737466c21c0c07ebab))
* **agents:** restore OPERATING-STANDARD reference as plain text ([f0dd35e](https://github.com/by-openclaw/doc-platform-core/commit/f0dd35eb8af100d8c4926a7d9e3ef98ccdbd8e4e))

## [0.6.0](https://github.com/by-openclaw/doc-platform-core/compare/v0.5.1...v0.6.0) (2026-04-11)


### Features

* add ADR-0032, ADR-0033, ADR-0034 ([424bd4b](https://github.com/by-openclaw/doc-platform-core/commit/424bd4beb7640ec38851e6b6d00030104e02c08a))


### Bug Fixes

* ADR-0027 update bootstrap for multi-WAN + svc-rune SSH ([dc201ee](https://github.com/by-openclaw/doc-platform-core/commit/dc201ee69eab36bf0c0635c2ff8f7881e7f71f26))
* ADR-0032 add subnet masks, inter-VLAN routing, SDN architecture ([e12eeb4](https://github.com/by-openclaw/doc-platform-core/commit/e12eeb40eb491b9b3b9c92ba124128599c06b775))
* ADR-0032 corrected VLAN scheme — prod 1001+, test 2001+ ([aa523a8](https://github.com/by-openclaw/doc-platform-core/commit/aa523a8cbcd2fb07380cdfba5fffc19a408b2a1f))
* ADR-0033 NOPASSWD forbidden, add svc-ansible account ([13f16d7](https://github.com/by-openclaw/doc-platform-core/commit/13f16d76cfa2cd60ede3cf3e4a04db48c13380d0))
* ADR-0033 remove application-specific references, add GDPR ([9ae3799](https://github.com/by-openclaw/doc-platform-core/commit/9ae3799c516a5e785398f915a9c226492ae32d04))
* **adr:** resolve 3 critical audit findings ([e60e05c](https://github.com/by-openclaw/doc-platform-core/commit/e60e05cefce9f47cff8d04e08b11bedecbbcf4cf))
* **docs:** canonical FQDN pattern, pin Docker images, fix RAM, standardize org placeholder ([85b6b2c](https://github.com/by-openclaw/doc-platform-core/commit/85b6b2c488194ceecf68da6c7e7362fb704a41dd))
* **docs:** canonical FQDN pattern, pin Docker images, fix RAM, standardize org placeholder ([4e179cf](https://github.com/by-openclaw/doc-platform-core/commit/4e179cfe0db2e1ec5cb93b5cce90aa6708b25f5a))
* **network:** vmbrOOB uses VLAN 300 (10.1.0.0/24) not 10.6.224.x ([388d0aa](https://github.com/by-openclaw/doc-platform-core/commit/388d0aa731e395d4db925554293ae10fbf24cef8))
* revert Redis image to redis:7.2.7-alpine (BSD-3) — 7.4+ is RSALv2/SSPL (R-25) ([ddd532d](https://github.com/by-openclaw/doc-platform-core/commit/ddd532d2daca03b9f8799898f074fec187f40f7d))
* **sdn:** replace env-prefixed VNet names with agnostic standard ([e2cdc70](https://github.com/by-openclaw/doc-platform-core/commit/e2cdc70b73d32e3550f1207a3405f85a3b6ea254))

## [Unreleased]

### Changed

* **ADR-0015** — Status updated to "Partially Implemented"; SDN zone `poc` + VNets deployed 2026-04-03; compliance table updated to reflect live state

---

## [0.5.1](https://github.com/by-openclaw/doc-platform-core/compare/v0.5.0...v0.5.1) (2026-03-30)


### Bug Fixes

* correct pull_request trigger in project-board-sync workflow ([f94e64d](https://github.com/by-openclaw/doc-platform-core/commit/f94e64de4056f4ec0fa4ae08049cf0a0958ae29b))

## [Unreleased]

### Changed

- `docs/adr/0015-network-vlan-architecture.md`: locked Proxmox SDN naming standard — zone = environment, VNet names environment-agnostic (`mgmt`, `dmz`, `svc`), removed stale `vmbrPOC` concept from active docs
- `docs/adr/0015-network-vlan-architecture.md`: added storage rule — `poc-iso` is the only valid storage target for PoC ISO / vztmpl media; never use `local`, `local-lvm`, or thin-LVM
- `docs/status.md`, `docs/raid.md`: replaced stale `vmbrPOC` blockers with SDN zone/VNet language
- `AGENTS.md`, `CLAUDE.md`: added explicit agent rule to stop inventing env-prefixed VNet names

### Added

- `docs/adr/0014-certificate-strategy.md`: Certificate strategy — LE for public, step-ca for internal, no self-signed, root CA in cloud-init mandatory
- `docs/adr/0015-network-vlan-architecture.md`: VLAN architecture — active VLANs 300/310/320/330/400/410, VLAN 340/350 removed from Arista trunk, OPNsense alias-first firewall rule standard
- `docs/adr/0016-vault-kv-path-convention.md`: Vault KV path convention — `secret/{env}/{service}/{key}`, no root-level paths, per-service least-privilege policies
- `docs/adr/0017-logging-standard.md`: Logging standard — Prometheus+Grafana+Loki on vm-observability-poc-01, 30d log / 90d metric retention, Zabbix Phase 2 only
- `docs/adr/0018-centralized-database-strategy.md`: Centralized DB strategy — single vm-postgres-poc-01 instance, per-service databases, single Redis for cache only
- `docs/adr/0019-git-workflow-approval-process.md`: Git workflow — trunk-based, Conventional Commits enforced, agents open PRs only, no force-push to main

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
