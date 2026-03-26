# `{{REPO_NAME}}`

> **Scope:** `{{SCOPE}}` | **Component:** `{{COMPONENT}}` | **Status:** `{{STATUS}}`  
> **GitLab:** `by-systems/{{SCOPE}}/{{REPO_NAME}}` | **License:** `{{LICENSE}}`

<!-- One or two sentences. What does this repo do and why does it exist? -->
{{SHORT_DESCRIPTION}}

---

## Table of Contents

- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Getting Started](#getting-started)
- [Configuration](#configuration)
- [CI/CD](#cicd)
- [Environments](#environments)
- [Security](#security)
- [Contributing](#contributing)
- [Architecture](#architecture)
- [License](#license)

---

## Overview

<!-- Expand the short description. What problem does this solve? Who uses it? How does it fit into the BY-SYSTEMS platform? -->

{{FULL_DESCRIPTION}}

**Platform tier:** `{{TIER}}` _(Tier 1 — Base Platform / Tier 2 — Module: {{MODULE_NAME}})_

---

## Prerequisites

| Requirement | Version | Notes |
|---|---|---|
| Docker | `>=24` | Container runtime |
| `{{PRIMARY_LANGUAGE}}` | `{{LANGUAGE_VERSION}}` | _(remove if not applicable)_ |
| HashiCorp Vault | Access required | Secrets management |
| GitLab Runner | Self-hosted | CI/CD execution |

Optional (for local dev):

- VSCode + devcontainer extension
- `kubectl` / `k9s` (if deploying to k3s)
- `terraform` CLI (if IaC repo)
- `ansible` (if Ansible repo)

---

## Getting Started

### Clone

```bash
git clone git@gitlab.by-systems.internal:by-systems/{{SCOPE}}/{{REPO_NAME}}.git
cd {{REPO_NAME}}
```

### Dev environment (devcontainer)

```bash
# Open in VSCode → "Reopen in Container"
# Or manually:
docker compose -f .devcontainer/docker-compose.yml up -d
```

### Local run

```bash
# 1. Copy and fill environment config
cp .env.example .env

# 2. Pull secrets from Vault (if applicable)
vault kv get -format=json {{SCOPE}}/{{COMPONENT}}/dev > .vault-secrets.json

# 3. Start service
{{START_COMMAND}}
```

---

## Configuration

All runtime configuration is managed via:

1. **Vault** — secrets (credentials, API keys, TLS certs)
2. **Environment variables** — non-sensitive config (see `.env.example`)
3. **Helm values** — K8S deployment config (see `charts/values.yaml`)

### Key environment variables

| Variable | Required | Default | Description |
|---|---|---|---|
| `{{VAR_1}}` | ✅ | — | {{VAR_1_DESC}} |
| `{{VAR_2}}` | ✅ | — | {{VAR_2_DESC}} |
| `{{VAR_3}}` | ⬜ | `{{VAR_3_DEFAULT}}` | {{VAR_3_DESC}} |

### Vault secret paths

```
{{SCOPE}}/{{COMPONENT}}/dev/     → development secrets
{{SCOPE}}/{{COMPONENT}}/staging/ → staging secrets
{{SCOPE}}/{{COMPONENT}}/prod/    → production secrets
```

---

## CI/CD

Pipeline defined in `.gitlab-ci.yml`.

| Stage | Tool | Description |
|---|---|---|
| `lint` | commitlint, shellcheck, yamllint | Code style and commit format |
| `test` | _(project-specific)_ | Unit and integration tests |
| `scan` | Trivy, Gitleaks, OWASP DC | Security and vulnerability scan |
| `build` | Kaniko | Rootless container image build |
| `deploy` | _(project-specific)_ | Deploy to target environment |
| `release` | release-please | Semver tagging + CHANGELOG |

Container image: `registry.by-systems.internal/{{SCOPE}}/{{COMPONENT}}:{tag}`

---

## Environments

| Environment | Branch | FQDN | Notes |
|---|---|---|---|
| `dev` | `feat/*`, `fix/*` | `{{COMPONENT}}.dev.by-systems.internal` | Auto-deploy on push |
| `staging` | `main` | `{{COMPONENT}}.staging.by-systems.internal` | Auto-deploy on merge |
| `prod` | `v*` tag | `{{COMPONENT}}.by-systems.internal` | Manual gate |

All services use Traefik as reverse proxy and internal TLS via `step-ca` (`*.by-systems.internal`).

---

## Security

- **Secrets:** HashiCorp Vault — never commit credentials or API keys
- **Git signing:** Ed25519 GPG (Vault-backed, annual rotation)
- **Container scanning:** Trivy runs on every pipeline
- **Secret scanning:** Gitleaks (pre-commit + CI)
- **IaC scanning:** Checkov (Terraform / Ansible / K8S manifests)
- **Compliance:** Wazuh SIEM + OpenSCAP (ISO 27001, NIS2)

For vulnerability reports, open a confidential issue with label `type:security`.

---

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for:

- Branch naming and commit conventions
- MR/PR workflow
- Code review process
- ADR process for significant decisions

---

## Architecture

<!-- Link to diagrams or embed a simple ASCII/PlantUML diagram here -->

Diagrams: [`docs/diagrams/`](docs/diagrams/)  
ADRs: [`docs/adr/`](docs/adr/)

```
<!-- Example: replace or remove -->
[Client] → [Traefik TLS] → [{{COMPONENT}}] → [PostgreSQL / Vault / NetBox]
```

Related services in NetBox: `https://netbox.by-systems.internal/dcim/services/?name={{COMPONENT}}`

---

## License

`{{LICENSE}}` — see [LICENSE](LICENSE)

<!-- For internal / proprietary repos: -->
<!-- © BY-SYSTEMS. All rights reserved. Internal use only. -->
