# ADR 0002 — Repository Structure and Naming Conventions

**Date:** 2026-03-26
**Status:** Accepted
**Scope:** All `by-openclaw` GitHub org repositories

---

## Context

BY-SYSTEMS is building an internal DevOps PoC platform. We need a consistent, scalable repository structure that:
- Mirrors folder structure across repos (config, scripts, runbooks, security, log)
- Handles tool upgrades including breaking API changes
- Separates documentation, platform setup, and application services clearly
- Supports CI/CD templates shared across all service repos

---

## Decision

### Repository Categories

| Category | Naming Pattern | Purpose |
|---|---|---|
| Documentation | `doc-{scope}` | Docs only — no code, no pipelines |
| Platform setup | `platform-{scope}` | Tool configs, runbooks, security, scripts |
| Infrastructure as Code | `ansible-{scope}`, `tf-{scope}` | Runnable IaC with their own CI |
| K8S workloads | `k8s-{scope}` | Manifests and Helm charts |
| Services | `svc-{name}` | Application services — one repo per service |
| CI Templates | `ci-templates` | Shared GitLab CI pipeline templates |

### Monorepo vs Per-Repo Rules

**Monorepo** (`platform-setup`) for:
- Tool configurations
- Install scripts
- Security hardening
- Runbooks
- Log configs

**Separate repo** for:
- Runnable IaC (Ansible, Terraform) — needs own versioning + pipeline
- Application services (`svc-*`) — own lifecycle, CI/CD, releases
- Documentation (`doc-*`) — no code, separate concern

**Service docs stay in the service repo** — `README.md`, architecture, runbooks, API docs. `doc-platform-core` covers platform-level docs only.

### Platform Setup Repo Structure (`platform-setup`)

Every tool follows the same internal folder structure:

```
platform-setup/
├── tools/
│   ├── {tool-name}/
│   │   ├── config/          # Tool configuration files
│   │   ├── scripts/         # Install and maintenance scripts
│   │   ├── security/        # Hardening, certs, firewall rules
│   │   ├── runbooks/        # Operational procedures
│   │   ├── log/             # Log config and rotation
│   │   └── docker-compose.yml
├── infra/
│   ├── proxmox/
│   ├── network/
│   └── dns/
├── automation/
│   ├── ansible/
│   └── terraform/
├── k8s/
│   ├── namespaces/
│   ├── helm/
│   └── manifests/
└── .gitlab-ci.yml
```

### Handling Breaking API Changes

Version folders inside the tool directory:

```
tools/netbox/
├── v3/          # Previous version — kept until migration complete
├── v4/          # Current version
└── README.md    # Documents active version + migration notes
```

Git tags mark last-known-good states before upgrades: e.g. `netbox-3.7-stable`.

### Service Repo Structure (`svc-*`)

```
svc-{name}/
├── src/                  # Application source code
├── tests/
├── k8s/ or helm/         # Deployment manifests
├── docs/                 # Service-specific docs (arch, runbooks, API)
│   ├── architecture.md
│   ├── runbook.md
│   └── api.md
├── .gitlab-ci.yml        # Includes from ci-templates
├── Dockerfile
└── README.md
```

### CI Templates Pattern

```yaml
# svc-*/. gitlab-ci.yml
include:
  - project: by-openclaw/ci-templates
    file: .gitlab-ci/build.yml
  - project: by-openclaw/ci-templates
    file: .gitlab-ci/deploy.yml
  - project: by-openclaw/ci-templates
    file: .gitlab-ci/security.yml
```

### End-to-End Provisioning Flow

```
NetBox (source of truth)
  └── new VM/device/service registered
        └── webhook → GitLab CI triggered
              └── ansible-platform runs provisioning playbook
                    └── OS + base config applied
                          └── K8S namespace created
                                └── svc-* pipeline deploys service
                                      └── NetBox status updated
```

---

## Consequences

- All repos follow predictable folder layouts — easy navigation
- Breaking changes in tools are handled with version subfolders, not repo forks
- Service docs live with service code — no cross-repo reference drift
- CI templates enforced centrally — consistent pipelines, no copy-paste
- `doc-platform-core` stays docs-only — agents know never to add code here

---

## References

- [Naming conventions](../naming-convention.md)
- [NetBox source of truth](../netbox.md)
- [Platform stack](../stack.md)
