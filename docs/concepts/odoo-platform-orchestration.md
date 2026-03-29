# Concept: Odoo → Platform Orchestration

**Status:** Concept — not scheduled  
**Date:** 2026-03-29  
**Author:** @yboujraf + Rune  

---

## Vision

**Self-hosted odoo.sh.** One (or a few) Proxmox nodes running many Odoo instances as lightweight LXC containers. Git branch = environment. `git push` → auto-deploy. Dev / test / staging / prod all on the same small hardware, isolated per container. BY-SYSTEMS is the SaaS provider — customers manage their instance from an Odoo portal account.

Reference products (study, don't copy):
- **odoo.sh** — git-driven, branch = env, SSH per container, runbot per branch
- **Cloudpepper** — white-label, backup schedules, staging neutralisation, partner API

### Architecture: LXC over VMs

Not one VM per customer. One **LXC container per Odoo instance** — much smaller footprint, many per node.

```
Proxmox node
├── lxc-odoo-customer-a-prod     (512MB RAM, 2 vCPU)
├── lxc-odoo-customer-a-staging  (512MB RAM, 1 vCPU)
├── lxc-odoo-customer-b-prod     (1GB RAM, 2 vCPU)
├── lxc-odoo-customer-c-dev      (256MB RAM, 1 vCPU)
└── ...N more
     shared: PostgreSQL LXC (one per node, per-instance DB)
     shared: Traefik LXC    (reverse proxy, TLS termination)
     shared: MinIO           (filestore — S3 backend for all instances)
```

### Git-Driven Environments (odoo.sh model)

```
GitLab repo: customer-a-odoo
├── branch: production   → lxc-odoo-customer-a-prod
├── branch: staging      → lxc-odoo-customer-a-staging
└── branch: feature-xyz  → lxc-odoo-customer-a-dev (ephemeral)

git push origin staging
→ GitLab CI pipeline triggers
→ ansible-platform deploys to lxc-odoo-customer-a-staging
→ odoo-install scripts run inside LXC (idempotent)
→ staging neutralised automatically on first deploy
```

---

## Full Flow

```
Customer (Odoo Portal)
    1. Selects template or custom config (CPU, RAM, disk, Odoo version, addons)
    2. Quotation generated (Odoo Sales)
    3. Order confirmed → payment processed (Odoo Accounting / Payment)
    4. Odoo custom module fires: POST /api/platform/provision
           ↓
    Platform Orchestration API  ← the missing piece (see below)
           ↓                    ↓
        NetBox               Event queue / webhook
     allocates IP,           triggers pipeline
     creates VM record,
     assigns customer tag
           ↓
        Terraform
     provisions VM from template (Proxmox)
           ↓
        ansible-platform
     hardens OS, configures Odoo instance (reuses odoo-install scope)
           ↓
    Instance ready
     → customer portal updated: URL, credentials, status
     → metrics exposed: CPU, RAM, disk, uptime
     → actions available: restart, stop, backup, restore
```

---

## Layers & Responsibilities

| Layer | Component | Role |
|---|---|---|
| Business | Odoo (Sales + Accounting + Portal) | Order lifecycle, payment, customer self-service |
| Connector | `odoo-platform` custom module | Translates business events → orchestration API calls |
| Orchestration API | Platform API (TBD) | Single entry point for all provisioning requests. Owns state machine. |
| CMDB | NetBox | Source of truth: IP allocation, VM record, customer assignment |
| Provisioning | Terraform (infra-terraform-proxmox) | VM creation from template |
| Configuration | ansible-platform + odoo-install scope | OS hardening, Odoo install, cert, nginx/Traefik, DNS |
| Observability | Metrics endpoint → Odoo portal | CPU, RAM, disk, uptime per instance |

---

## The Missing Piece: Platform Orchestration API

Everything else exists (partially). The gap is a **lightweight orchestration API** that:

- Receives provisioning requests from Odoo
- Coordinates NetBox → Terraform → Ansible in order
- Tracks instance state (provisioning / running / stopped / failed)
- Exposes instance status + metrics back to Odoo
- Handles lifecycle actions: restart, stop, backup, restore

**Candidate:** FastAPI service, deployed as a VM or container. Talks to:
- NetBox API (allocate/release resources)
- GitLab CI (trigger Terraform + Ansible pipelines)
- Proxmox API (direct VM lifecycle: restart/stop)
- MinIO S3 (backup/restore operations)

---

## Odoo Custom Module — `odoo-platform`

A dedicated Odoo module (not a hack on Sales). Responsibilities:

- **Provision trigger:** on SO confirmation, POST to orchestration API
- **Status polling:** periodic job syncs instance state back to Odoo
- **Portal views:** customer sees their instance(s), URL, status, metrics
- **Actions:** restart/stop/backup/restore → POST to orchestration API
- **Billing hooks:** usage-based billing if needed (future)

Follows ADR-0007: idempotent, present/absent, dry-run/validation before any API call.

---

## Reuse from odoo-install

`by-systems/odoo-install` is the reference for Odoo instance configuration:
- nginx/Traefik proxy setup
- Certbot DNS-01 (Cloudflare)
- DNS sync (A/AAAA/CNAME via Cloudflare API)
- PostgreSQL provisioning
- Fail2ban, GeoIP, log rotation
- instances.yml pattern → drives per-instance config

The ansible-platform role for Odoo **wraps odoo-install** — same scripts, driven by Ansible variables. No duplication, no divergence.

---

## Reference: Cloudpepper.io

Cloudpepper is the closest existing product to this vision — Odoo hosting platform with portal, backups, staging, Git deploy, white-label, and API. **SaaS, not self-hosted.** Study their UX and feature set; build the equivalent on sovereign infrastructure.

Key patterns to adopt:
- **Staging neutralisation:** clone prod → auto-disable outbound email + cron jobs on clone. Prevents accidental customer emails during testing. Must-have.
- **White-label portal:** customer logs in at `portal.by-systems.be`, sees only their instances. No BY-SYSTEMS branding leak.
- **Git autodeploy:** GitLab push → pipeline → module upgrade on instance. Cloudpepper does this; we have GitLab CI already.
- **Backup retention policies:** daily/weekly/monthly, configurable per instance. MinIO ILM handles tiering; orchestration API handles scheduling.

### Staging Neutralisation — Detail

When cloning prod → staging, the following are toggled automatically via Odoo API / script:

| Resource | Action |
|---|---|
| `ir.cron` (all scheduled actions) | `active = False` — no background jobs run |
| `ir.mail_server` (outbound SMTP) | Disabled or redirected to catch-all mailbox |
| Payment acquirers | Switched to test/sandbox mode |
| External webhooks / API integrations | Disabled |

Clone has full prod data. Cannot reach real customers. Safe to test modules, upgrades, data migrations. Scriptable and reversible.

---

## Customer Portal Features (MVP)

| Feature | Source |
|---|---|
| Instance URL | Provisioned by pipeline, stored in NetBox |
| Status (running/stopped/error) | Orchestration API → Odoo |
| CPU / RAM / disk metrics | Proxmox API → Orchestration API → Odoo |
| Restart / Stop | Portal action → Orchestration API → Proxmox API |
| Backup now | Portal action → Orchestration API → MinIO S3 |
| Restore from backup | Portal action → Orchestration API → MinIO S3 |
| Staging clone (neutralised) | Portal action → clone VM → disable email/cron → notify customer |
| Upgrade Odoo version | Future (post-MVP) |
| Git autodeploy | Future — GitLab push → pipeline → module upgrade |

---

## Business Model: BY-SYSTEMS as SaaS Provider

BY-SYSTEMS operates as a sovereign SaaS hosting platform under its own brand. Odoo portal is the customer-facing product. The infra stack is the engine — invisible to customers.

### Three Tiers

| Tier | Who | Portal | What they see |
|---|---|---|---|
| **Admin** | BY-SYSTEMS team | Internal admin | Full infra, all customers, all instances, billing |
| **Customer** | End client | `portal.by-systems.be` | Their instances only — metrics, actions, billing |
| **Partner** | Odoo reseller | `portal.theirbrand.be` (CNAME to BY-SYSTEMS infra) | Their clients' instances under their brand |

### Partner / Reseller Tier

Partners get a white-label sub-portal under their own domain and branding. Their clients never see BY-SYSTEMS — they see the partner's brand. BY-SYSTEMS provides:
- Infra + provisioning
- Backup + monitoring
- Orchestration API access (scoped to partner's instances)

Partner manages their own client relationships, billing, support.

---

## Open Questions (for later)

- Multi-tenant isolation: one VM per customer vs shared + containerised?
- Billing model: flat monthly vs usage-based?
- SLA tiers: which tier gets which resource guarantee?
- Orchestration API: build vs adopt (Temporal, Prefect, or simple FastAPI)?
- GitLab CI as trigger vs direct Terraform API call?

---

## Status

**Not scheduled.** Prerequisites:
1. NetBox deployed and validated ← in progress
2. Terraform VM provisioning stable ← in progress
3. ansible-platform hardening role complete ← in progress
4. Platform Orchestration API — not started
5. `odoo-platform` Odoo module — not started

Revisit when items 1-3 are stable.
