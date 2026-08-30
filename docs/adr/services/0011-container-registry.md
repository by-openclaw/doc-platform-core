# services/0011 — Container Registry (Harbor)

**Status:** Draft (documents the deployed service)
**Date:** 2026-08-30
**Scope:** Platform OCI registry — engine, storage, auth, pull-through caching, CI integration, scanning. Closes the review gap: the registry was deployed (2026-08) with no owning ADR.
**Related:** `services/0009-object-storage`, `services/0004-database-strategy`, `security/0003-hardening §4` (Trivy gate), `security/0004-certificate-strategy`, `identity/0001-authentication`

---

## Context

The platform builds and consumes OCI images (Docker today; k3s/Helm OCI tomorrow) and must not depend on Docker Hub availability/rate limits. Nexus CE was rejected for registry duty (capped); Harbor is the OCI-native choice.

## Decision

### 1. Engine — Harbor (pinned release) on `lxc-harbor-01`

Installed from the pinned online installer (never `:latest`), docker-compose stack. **TLS terminates at Traefik** (`harbor.{domain}`, internal-only route — VPN/LAN + CI + cluster reach it; no public exposure); Harbor serves plain HTTP behind it, `external_url` carries https.

### 2. State placement (stateless-ish service)

| State | Where |
|---|---|
| Image blobs | **SeaweedFS S3** (`harbor-registry` bucket, scoped identity — `services/0009`) |
| Metadata | **cluster PostgreSQL** (`harbor` DB, `sslmode=verify-full`) |
| Job queue/cache | Harbor's **own bundled Redis** (isolated by design — not the cluster Redis) |
| Local disk | logs + Trivy cache only |

### 3. AuthN/Z

- **Authentik OIDC** (`auth_mode=oidc_auth`), auto-onboard; group claim maps `harbor-admins` → Harbor administrators. No local users beyond the sealed local admin (fabric/Vault).
- **CI access via a scoped system robot** (`robot$gitlab-ci`): push/pull on the private `gitlab` project, pull-only on the cache. Robot secret: fabric → Vault (`prod/harbor/robot-gitlab`); exposed to pipelines as GitLab **instance CI variables** (`HARBOR_HOST/ROBOT_USER/ROBOT_TOKEN`, token masked) seeded by a `gitlab-rails` reconciler.

### 4. Docker Hub replacement — pull-through proxy cache

Project `dockerhub` proxies `docker.io` (first pull caches; later pulls local; rate-limit-proof; cached images are Trivy-scannable). Consumers pull `harbor.{domain}/dockerhub/<image>` or mirror `docker.io` at the runtime level. Additional upstreams (ghcr, quay) are config entries.

### 5. Scanning

Trivy runs in-registry; the CI image gate itself is `security/0003 §4` (zero critical+high) — registry scanning complements, never replaces, the CI gate.

## Consequences

- Registry loss ≠ data loss: blobs in S3, metadata in the cluster DB — a rebuilt guest re-attaches to both.
- One robot per consumer system, least-privilege; rotation = recreate (Harbor shows a robot secret once).
- The service is the platform's reference "complete chain" template (`roles/harbor/README.md`).

## Revision triggers

Harbor replaced; public exposure required; per-project robot model needed; image signing (Notary/cosign) adopted; k3s consuming the registry (may add HA requirements).

## CISO mapping

| Framework | Controls |
|---|---|
| ISO 27001:2022 | A.8.8 (vulnerability mgmt — Trivy), A.5.15/A.8.3 (access — OIDC + scoped robots), A.8.12 (leakage — private projects) |
| NIS2 | Art. 21(2)(d) (supply chain — pinned engine, cached upstreams), 21(2)(e) |
