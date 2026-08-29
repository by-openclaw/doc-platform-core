# services/0008 — Object Storage (S3)

**Status:** Draft
**Date:** 2026-08-29
**Scope:** Platform S3-compatible object storage — which engine, bucket/identity model, endpoint exposure, consumer contract. Closes the gap found in the ADR↔implementation review: object storage was core infrastructure with no owning ADR.
**Related:** `security/0001-secret-storage`, `security/0004-certificate-strategy`, `infra/0008-backup-strategy`, `naming/0001-infra §9`, `services/0004-database-strategy` (the analogous shared-datastore contract)

---

## Context

Multiple services need S3 object storage: PBS (S3 datastore), Harbor (image blobs), GitLab (artifacts/LFS/uploads/registry), Nextcloud (primary storage), Vault DR (raft snapshots), JumpServer (session replays). MinIO was the original engine; upstream archived its open-source COTS path (EOL for our purposes) and it was **replaced by SeaweedFS**. The bucket/identity pattern was established in Ansible but never contracted.

## Decision

### 1. Engine — SeaweedFS

SeaweedFS (`lxc-seaweedfs-01`) is the platform S3 engine — Apache-2.0, S3-compatible API, path-style addressing. MinIO is decommissioned (never redeploy; `lxc-minio-01` references are stale).

### 2. Endpoint

One S3 endpoint per deployment: `s3.{domain}` — TLS at Traefik (LE zone wildcard per `security/0004`), internal exposure (split-DNS; VPN/LAN reach). Consumers use **path-style** requests (`https://s3.{domain}/{bucket}/…`) with SigV4.

### 3. Bucket + identity model — per service, least privilege

- **One bucket (or bucket set) per service**, named after the service: `harbor-registry`, `gitlab-artifacts`, `pbs-datastore`, …
- **One scoped S3 identity per service** — access key limited to that service's buckets only. No shared credentials, no anonymous buckets.
- Both are provisioned by the reusable Ansible role `seaweedfs_bucket` (idempotent ensure of bucket + identity), included by each service role — the same shared-role pattern as `postgres_db` and `traefik_route`.
- Credentials follow `security/0001`: generated once → controller fabric file (`infra-seaweedfs-{service}.json`) → Vault mirror at the KV path convention.

### 4. Consumer contract

A service consuming S3 declares: bucket name(s), identity name, and endpoint — all via role defaults consuming `{{ platform_domain }}` (no hardcoded endpoints). Blob data lives in S3; service-local disk holds only caches/logs. Decommission reverses it: the `service_decommission` catalog removes the bucket (archive-before-destroy per platform rule) and the identity.

### 5. Backup interplay

S3 payload data is backed up per `infra/0008-backup-strategy`. Two locked gotchas: the PBS host must **not** back itself up into its own S3 datastore (circular; PBS's DR copy goes to NFS), and SeaweedFS's own volume data is backed up at the guest level like any LXC.

## Consequences

- Object storage finally has an owning contract; new services get S3 by including one role with three variables.
- Per-service identities keep blast radius to one service's buckets.
- Engine swap (SeaweedFS → other) = revise this ADR + the `seaweedfs_bucket` role internals; consumers are unaffected (S3 API + role interface stable).

## Revision triggers

Revise when: SeaweedFS is replaced or forked away; a consumer needs cross-service bucket access; multi-node S3 (EC/replication) is introduced; a public (internet-exposed) bucket use case appears.

## CISO mapping

| Framework | Controls covered |
|---|---|
| ISO 27001:2022 | A.5.15/A.8.3 (access restriction — per-service identities), A.8.12 (data leakage — no anonymous buckets), A.8.13 (backup — infra/0008 interplay) |
| NIS2 | Art. 21(2)(d) (supply chain — OSS engine, pinned), 21(2)(e) (systems security) |
| GDPR | Art. 32(1)(a) (TLS in transit via Traefik), Art. 5(1)(f) (integrity/confidentiality) |
