# infra/0002 — Platform Charter

**Status:** Draft
**Date:** 2026-04-13 (supersedes flat ADR-0006, 2026-03-28)
**Scope:** Build-order contract — the platform is built in strict sequential layers. Each layer must be documented, tested, and signed off before the next begins.
**Related:** `infra/0001-platform-stack`, `infra/0003-terraform-standard`, `identity/0004-os-accounts`, `security/0001-secret-storage`, `security/0003-hardening`, `services/*` (future)

---

> **Rule #1:** Nothing proceeds to the next layer until the current layer is documented, tested, and signed off.
> **Rule #2:** Every decision lives in an ADR. Every finding lives in RAID + GitHub issue + project board.
> **Rule #3:** AI agents assist. They do not decide. They do not skip layers.

---

## Context

The platform requires a strict build order. Deploying services before the foundation is stable creates cascading failures, untraceable dependencies, and security gaps. A formal layer model governs sequencing and acceptance criteria — each layer must pass its gate before the next layer begins.

## Decision

The platform is built in strict sequential layers. No layer may begin until the previous layer passes its acceptance criteria, is documented, and is signed off by @yboujraf.

```
┌─────────────────────────────────────────────────────────────────┐
│  Layer 0 — Standards & Templates                               │
│  Define once. Apply forever. No exceptions.                     │
│  Acceptance: ADRs written, standards files exist, templates set │
├─────────────────────────────────────────────────────────────────┤
│  Layer 1 — Proxmox Base                                        │
│  Cloud-init template finalized. VM/LXC baseline locked.        │
│  Acceptance: any VM provisioned = same baseline, zero manual    │
├─────────────────────────────────────────────────────────────────┤
│  Layer 2 — Vault + step-ca                                     │
│  Secrets management + internal PKI.                            │
│  Acceptance: Vault cluster up, PKI + KV mounted, policies set, │
│              step-ca root CA distributed to all VMs            │
├─────────────────────────────────────────────────────────────────┤
│  Layer 3 — Identity (Authentik + federation)                   │
│  All platform users from Authentik. SSH via CA. No static keys.│
│  Acceptance: Authentik HA live, federation optional, SSH via   │
│              Vault-issued certificates, no plaintext keys      │
├─────────────────────────────────────────────────────────────────┤
│  Layer 4 — Storage (Synology + MinIO)                          │
│  NFS shares, S3 object store, permissions managed in code.     │
│  Acceptance: lib-synology-dsm tested, CI green, shares         │
│              provisioned via Ansible/Terraform                 │
├─────────────────────────────────────────────────────────────────┤
│  Layer 5 — Platform Services                                   │
│  NetBox, GitLab CE, monitoring, logging, backup, email, etc.   │
│  Acceptance: each service has runbook, CI gate, RAID entry,    │
│              monitoring target, backup policy                  │
└─────────────────────────────────────────────────────────────────┘
```

### Layer scope — what belongs where

**Layer 0 — Standards & Templates**
- All scoped ADRs in `docs/adr/{git,identity,naming,security,infra,services,lib}/`
- `OPERATING-STANDARD.md`
- Per-repo `CLAUDE.md` / `AGENTS.md` / `CONTRIBUTING.md`
- Document templates, issue templates, PR templates

**Layer 1 — Proxmox Base**
- Cloud-init template for every VM
- `ansible-platform/roles/hardening` applied to every base image
- VM hostname pattern per `naming/0001-infra`
- User accounts per `identity/0004-os-accounts` (pre-Authentik: local-only)

**Layer 2 — Vault + step-ca**
- HashiCorp Vault HA (2 instances) per `security/0001-secret-storage`
- step-ca root and intermediate CA per `security/0004-certificate-strategy`
- `ca-trust` Ansible role distributing step-ca root to every VM
- Vault policies per service, AppRole auth, audit logging to Loki

**Layer 3 — Identity**
- Authentik HA (2 instances) per `identity/0001-authentication`
- Authentik → Vault integration (Authentik accounts provisioned via identity-sync)
- SSH certificate authority via Vault PKI (no more static `authorized_keys` for `svc-*`)
- Optional upstream federation (EntraID, Google, LDAP) per org request

**Layer 4 — Storage**
- Synology NFS for backup destinations and shared platform state
- MinIO on-prem for Loki object backend and CI artifact retention
- `lib-synology-dsm` and Ansible roles tested and green before services depend on storage

**Layer 5 — Platform Services**
- GitLab CE self-hosted
- NetBox (CMDB source of truth)
- Prometheus + Grafana, Loki + Promtail
- Mailcow (Contabo-hosted, but provisioned after Layer 3 for SSO)
- Vaultwarden (human password manager)
- Traefik (edge reverse proxy) — straddles Layer 2 (TLS) and Layer 5 (routing)
- Per-service runbook in `platform-setup/tools/{service}/runbooks/`

## Parallel development exception

**Layer 4 library work** (`lib-synology-dsm`, `lib-opnsense`, Python and Ansible collections) may be developed in parallel with infrastructure layers. Library code has no runtime dependency on any deployed infrastructure — it ships as a package and is consumed by whatever layer needs it later.

This is the **only** parallelism exception. Infrastructure work is strictly sequential.

## Gating rules

- A layer is "done" when: **documented + tested + signed off by @yboujraf**. Not before.
- Any agent or contributor detecting a gap in a lower layer must **stop and raise a RAID item** before proceeding.
- No service in Layer 5 may be deployed before Layers 1–4 are complete.
- Gate evidence (documentation, test output, sign-off) is kept alongside the work — PR descriptions, runbook updates, commit references.
- Layer reopens are allowed (e.g. adding a new Vault policy after Layer 2 sign-off) but must not block upper layers.

## Agent rules (Rule #3 expanded)

- Agents **may** propose ADRs, draft content, run tests, open PRs, analyse compliance posture, and report findings
- Agents **may not** sign off on a layer transition, merge PRs to main, or bypass any gate
- Agents that detect a gate violation must halt and raise a RAID / issue; they do not "fix it and move on"
- @yboujraf is the sole gate authority — per `git/0001-workflow §Agent boundary`

## Consequences

- **Sequential, auditable build.** Trades speed for correctness. Suitable for ISO 27001 and NIS2 evidence — every transition has a documented gate.
- **Lib/tooling parallelism** is explicitly allowed so that Python libraries and Ansible collections don't block on infra deployment.
- **Gate reopening is explicit**, not implicit. Adding a new Vault policy post-Layer 2 doesn't re-trigger the gate; rewriting the Vault PKI mount does.
- **Agent boundary is enforced at three places**: workflow (`git/0001`), charter (this ADR), and per-scope ADRs that describe automated vs manual actions.

## Revision triggers

Revise when:
- A new layer is inserted (unlikely — 6 layers cover the platform architecture)
- The parallelism exception is extended to another category of work
- @yboujraf delegates gate authority (currently sole authority)
- A compliance framework forces explicit evidence requirements for specific layers (e.g. ISO audit requires a gate log per transition)

## CISO mapping

| Framework | Controls covered |
|---|---|
| ISO 27001:2022 | A.5.1 (policies), A.8.9 (configuration management), A.8.32 (change management — no layer advance without gate) |
| NIS2 | Art. 21(2)(a) (risk management — layered build reduces uncontrolled-change risk) |
