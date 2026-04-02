<!--
| Field        | Value            |
|--------------|------------------|
| Created      | YYYY-MM-DD       |
| Last updated | YYYY-MM-DD       |
| Updated by   | {agent or human} |
-->

# <img src="../../assets/logos/logo-{tool}.png" height="40" alt="{Tool} logo"> {Tool Name}

> **Source:** {upstream GitHub URL}
> **Image:** `{docker image reference}`
> **Version:** {tracked version}
> **License:** {SPDX} — {free / paid / commercial}
> **Owner:** BY-SYSTEMS
> **CISO:** NIS2 ✓/⚠/✗  |  ISO27001 ✓/⚠/✗  |  GDPR ✓/⚠/✗
> **Layer:** {N} — {Layer name}
> **VM:** `{vm-hostname-poc-01}`
> **Deploy order:** {N}
> **Created:** YYYY-MM-DD  |  **Updated:** YYYY-MM-DD  |  **By:** {name}

> ⚠️ **Logo disclaimer:** All product logos and trademarks are property of their respective owners.
> BY-SYSTEMS uses them solely for internal documentation and identification purposes.
> No affiliation with or endorsement by the respective trademark holders is implied.

---

## Contents

### Identity & Infrastructure
- [Identity](docs/identity.md) — VM FQDN, service URL, layer, deploy order
- [Provisioning](docs/provisioning.md) — vCPU, RAM, disk, cloud-init, OS
- [Environment variables](docs/env.md)
- [Dependencies](docs/dependencies.md) — upstream + downstream

### Networking & Access
- [Network](docs/network.md) — ports, VLANs, firewall, Traefik config
- [TLS](docs/tls.md) — cert source, trust model, no-warning guarantee
- [Credentials](docs/credentials.md) — Vault KV paths, rotation owners

### Operations
- [Setup](docs/setup.md) — install, config, integrations, health check
- [Hardening](docs/hardening.md) — security checklist
- [Backup](docs/backup.md) — volumes, schedule, backend, restore
- [Monitoring](docs/monitoring.md) — metrics endpoint, alerts, retention

### Governance
- [Compliance](docs/compliance.md) — NIS2 / ISO27001 / GDPR mapping
- [Licensing](docs/licensing.md) — SPDX, OSS vs Enterprise, feature matrix
- [References](docs/references.md) — upstream docs, ADRs, standards, deployment artifact
- [Community](docs/community.md) — channels, ecosystem, plugins catalog

### Terraform & Config
- [Terraform spec](terraform/README.md)
- [Config files](config/)
- [Runbooks](runbooks/)
- [Security](security/)
