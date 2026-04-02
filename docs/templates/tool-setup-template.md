# {TOOL_NAME} — Setup Runbook

<!--
  SCOPE GUARD
  ===========
  This is the operational setup runbook for {TOOL_NAME}.
  - One file per tool, in tools/{tool}/runbooks/setup.md
  - Produced from: doc-platform-core/docs/templates/tool-setup-template.md
  - Do NOT duplicate config from ADRs — reference them
  - Do NOT include credential values — reference Vault paths only
  - Keep current: update after every upgrade or config change
-->

**Tool:** {TOOL_NAME}
**Version:** {VERSION}
**Image:** {IMAGE}
**VM:** `{vm-name}.by-systems.be`
**Layer:** Layer {N} — {layer name} (ADR-0006)
**Owner:** @yboujraf
**Last updated:** YYYY-MM-DD

---

## 1. Image Decision

| Option | Image | Approach | Decision |
|---|---|---|---|
| Official | `{official-image}` | … | ✅ / ❌ |
| Alternative | `{alt-image}` | … | ✅ / ❌ |

**Decision:** Use `{chosen-image}`.

| Reason | Detail |
|---|---|
| … | … |

---

## 2. Services Architecture

```
{ASCII diagram showing all containers, dependencies, external services}

Example layout:
  Traefik (DMZ)
    └── {tool} container :{port}
          ├── PostgreSQL (centralized)
          ├── Redis (centralized)
          └── S3 (Contabo)
```

| Container | Image | VM | Purpose |
|---|---|---|---|
| {tool} | `{image}:{version}` | `{vm}` | … |
| … | … | … | … |

---

## 3. Pre-requisites

Before starting this service:

- [ ] Dependencies from ADR-0006 layer model satisfied
- [ ] `vm-postgres-poc-01` running (if using centralized PG)
- [ ] `vm-redis-poc-01` running (if using centralized Redis)
- [ ] Database `{db_name}` created with user `svc_{tool}`
- [ ] Traefik running and routing configured
- [ ] Pi-hole resolves `{tool}.poc.by-systems.be`
- [ ] Vault running and secrets populated (see §4)
- [ ] step-ca root CA trusted on this VM (if using internal TLS)

### Database setup (if required)

```bash
docker exec -it postgresql psql -U postgres <<EOF
CREATE USER svc_{tool} WITH ENCRYPTED PASSWORD '${TOOL_DB_PASS}';
CREATE DATABASE {db_name} OWNER svc_{tool};
GRANT ALL PRIVILEGES ON DATABASE {db_name} TO svc_{tool};
EOF
```

---

## 4. Secrets (Vault KV paths)

Generate all secrets BEFORE first start. Store in Vault under `secret/poc/{tool}/`.

| Secret | Vault path | How to generate | Notes |
|---|---|---|---|
| DB password | `secret/poc/{tool}/db-password` | `openssl rand -hex 32` | … |
| … | … | … | … |

> Vault path convention: ADR-0016

---

## 5. Certificates (if required)

```bash
# Only if internal TLS is needed for this service
# Use step-ca for internal certs, LE (via Traefik) for external
```

Reference: ADR-0014 — Certificate Strategy

---

## 6. Environment File

File: `config/{tool}.env.example` — copy to `.env` on the VM, fill real values from Vault.

```bash
# =============================================================================
# {TOOL_NAME} — Environment Variables
# Image: {image}:{version}
# Ref: {upstream docs URL}
# =============================================================================

# ── General ──────────────────────────────────────────────────────────────────
TZ=Europe/Brussels

# ── Network ──────────────────────────────────────────────────────────────────
{TOOL}_BIND_IP=10.1.3.{X}
{TOOL}_HOST={tool}.poc.by-systems.be

# ── Database ─────────────────────────────────────────────────────────────────
# (if applicable)

# ── Secrets (from Vault) ─────────────────────────────────────────────────────

# ── SSO / OIDC (Authentik) — enable after Authentik deployed ─────────────────
# OIDC_CLIENT_ID=...
# OIDC_CLIENT_SECRET=${TOOL_OIDC_CLIENT_SECRET}
# OIDC_ISSUER=https://auth.poc.by-systems.be/application/o/{tool}/

# ── S3 Object Storage (Contabo) ──────────────────────────────────────────────
# S3_ENDPOINT=${S3_ENDPOINT}
# S3_ACCESS_KEY=${S3_ACCESS_KEY}
# S3_SECRET_KEY=${S3_SECRET_KEY}
# S3_BUCKET=by-poc-{tool}
```

---

## 7. Docker Compose

File: `config/docker-compose.yml`

```yaml
services:
  {tool}:
    image: {image}:{version}
    restart: unless-stopped
    ports:
      - "${TOOL_BIND_IP}:{port}:{port}"
    volumes:
      - {tool}-data:/data
    env_file:
      - .env
    healthcheck:
      test: ["CMD", "{healthcheck command}"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s
    logging:
      driver: json-file
      options:
        max-size: "50m"
        max-file: "5"

volumes:
  {tool}-data:

# External dependencies (NOT in this compose):
# PostgreSQL: vm-postgres-poc-01 (${POSTGRES_HOST}:5432)
# Redis: vm-redis-poc-01 (${REDIS_HOST}:6379)
```

---

## 8. Traefik Routing

Add to Traefik `dynamic/services.yml` on `vm-traefik-poc-01`:

```yaml
http:
  routers:
    {tool}:
      rule: "Host(`{tool}.poc.by-systems.be`)"
      service: {tool}
      tls:
        certResolver: letsencrypt

  services:
    {tool}:
      loadBalancer:
        servers:
          - url: "http://${TOOL_BIND_IP}:{port}"
```

Reference: ADR-0014 (TLS), ADR-0015 (network/VLAN)

---

## 9. Installation Steps

```bash
# 1. SSH into the VM
ssh by-systems@{vm-name}.by-systems.be

# 2. Create directories
sudo mkdir -p /opt/{tool}/config
cd /opt/{tool}

# 3. Copy compose + env
cp config/docker-compose.yml .
cp config/{tool}.env.example .env
# Edit .env — fill real values from Vault

# 4. Verify external dependencies (if applicable)
# Test DB connection
# Test Redis connection

# 5. Start
docker compose up -d

# 6. Check logs
docker compose logs -f {tool}

# 7. Verify health
docker compose ps
```

---

## 10. Post-Install Configuration

| Step | What | How |
|---|---|---|
| 1 | First login | … |
| 2 | Disable public access | … |
| 3 | Configure OIDC | Enable after Authentik deployed |
| … | … | … |

---

## 11. Upgrade Procedure

```bash
# 1. Backup data
# 2. Check upgrade notes for version X → Y
# 3. docker compose down
# 4. Update image tag
# 5. docker compose up -d
# 6. Verify health
```

---

## 12. Health Checks

| Check | Command | Expected |
|---|---|---|
| Web UI | `curl -s -o /dev/null -w "%{http_code}" https://{tool}.poc.by-systems.be` | `200` |
| API | `curl -s https://{tool}.poc.by-systems.be/api/health` | `{"status":"ok"}` |
| Docker | `docker inspect {tool} --format='{{.State.Health.Status}}'` | `healthy` |

---

## 13. Troubleshooting

| Problem | Cause | Fix |
|---|---|---|
| 502 Bad Gateway | Service not ready | Check `docker compose logs {tool}` |
| DB connection refused | PG not running or wrong creds | Verify `DB_HOST`, test with psql |
| … | … | … |

---

## References

- ADR-0006 — Platform Charter (layer model)
- ADR-0014 — Certificate Strategy
- ADR-0015 — Network VLAN Architecture
- ADR-0016 — Vault KV Path Convention
- ADR-0018 — Centralized Database Strategy
- `tools/{tool}/docs/identity.md` — VM FQDN, service URL
- `tools/{tool}/docs/provisioning.md` — VM specs
- `{upstream docs URL}`
