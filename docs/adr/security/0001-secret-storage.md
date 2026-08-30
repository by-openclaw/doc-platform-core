# security/0001 — Secret Storage

**Status:** Draft
**Date:** 2026-04-13 (supersedes flat ADR-0011, 2026-03-31 + flat ADR-0016, 2026-04-02)
**Scope:** Where platform secrets live, who reads them, and how they are organized. Covers two complementary tools: HashiCorp Vault (machine secrets) and Vaultwarden (human password manager).
**Related:** `identity/0003-machine-credentials`, `identity/0004-os-accounts §11`, `naming/0001-infra`, `infra/0005-environment-tiers` (future)

---

## Context

Platform secrets previously lived in scattered `.env` files and ad-hoc JSON blobs on the Rune VM. Without a binding storage decision:

- Credentials leak into git via missed `.gitignore` entries
- No audit trail — who read which secret, when?
- No rotation — stale credentials survive forever
- No encryption at rest
- Humans and machines share the same storage paths, blurring the security model

This ADR defines **two storage tools** with a hard boundary between them — HashiCorp Vault for machine secrets, Vaultwarden for human password management — and the path/policy conventions for each.

## Decision

### Two tools, one boundary

| Tool | Purpose | Kind of secrets | Consumers |
|---|---|---|---|
| **HashiCorp Vault** | Programmatic machine secret store | API tokens, SSH passphrases, service account credentials, `ansible_become_pass`, OIDC client secrets, database credentials, TLS private keys, webhook HMAC keys | Ansible, CI/CD, applications, service accounts — never humans browsing a UI |
| **Vaultwarden** | Human password manager | Operator passwords for web app logins (Grafana, NetBox, Proxmox WebUI, GitLab, Authentik admin UIs), personal TOTP backup codes | Humans via browser extension / mobile app |

**The boundary is strict:**

- Machine processes read from **Vault** via API (AppRole / token) — never from Vaultwarden
- Humans store web-app passwords in **Vaultwarden** — never in Vault
- SSH key passphrases for service accounts (`svc-rune-{env}`, `svc-ansible-{env}`) live in **Vault** — not Vaultwarden (per `identity/0004-os-accounts §2`)
- SSH key passphrases for human operators (`yboujraf`, `adm_yboujraf`) live in the operator's memory with **Vaultwarden** as personal backup — not Vault

Migrating a secret between tools requires an ADR amendment. The boundary is not soft.

### HashiCorp Vault — authoritative machine-secret store

Vault is deployed as a Layer 2 dependency (see future `infra/0002-platform-charter`). All platform tools must be able to reach Vault before they can run. Vault is HA (2 instances) to avoid a single point of failure.

#### KV v2 path convention

```
secret/{env}/{service}/{key}
```

| Segment | Values | Notes |
|---|---|---|
| `env` | `dev`, `test`, `staging`, `acc`, `prod`, `drp` | Environment tier — see `infra/0005-environment-tiers` (6 tiers, no `poc`) |
| `service` | Service short code | Matches the service token in `naming/0001-infra §5` (e.g. `authentik`, `gitlab`, `netbox`, `svc-rune`, `svc-ansible`) |
| `key` | Key name | Lowercase, hyphen-separated |

**Examples:**

```
secret/dev/authentik/admin-password
secret/prod/gitlab/admin-password
secret/prod/netbox/secret-key
secret/dev/postgres/superuser-password
secret/prod/traefik/cloudflare-api-token
secret/prod/svc-rune/ssh-passphrase
secret/prod/svc-ansible/ssh-passphrase
secret/prod/svc-ansible/become-pass
secret/prod/authentik/webhook-secret
```

**Rules:**

1. **No root-level paths.** `secret/authentik-password` is banned — every path has the 3-segment structure.
2. **Env is always explicit.** No shared secrets across envs — `secret/dev/...` and `secret/prod/...` are fully independent.
3. **Service short code matches NetBox service vocabulary** (see `naming/0001-infra §5` and `services/0003-netbox-cmdb` when deployed).
4. **Keys are lowercase, hyphen-separated.** No underscores, no CamelCase.
5. **Structured secrets (JSON blobs) are stored as single KV values** — individual fields accessible via `vault kv get -field=...`. Never split a single logical credential across multiple KV paths.

#### Per-service access policies

Each service has a dedicated Vault policy scoped to its own path prefix:

```hcl
# policy: gitlab-dev
path "secret/data/dev/gitlab/*" {
  capabilities = ["read"]
}
path "secret/metadata/dev/gitlab/*" {
  capabilities = ["list"]
}
```

**Rules:**

1. **No policy grants cross-service access.** `gitlab` cannot read `netbox` secrets, `svc-rune-dev` cannot read `svc-rune-prod`.
2. **Least privilege always.** Read is default; write is granted only to rotation workflows.
3. **AppRole auth method** for service accounts — per-env role_id + wrapped secret_id, rotation-ready.
4. **Root token revoked** immediately after init. Unseal keys in encrypted cold storage, never in the running cluster.
5. **Audit logging mandatory.** Every `vault kv read/write` goes to the audit log; logs forwarded to Loki (see future `infra/0006-logging` when refactored from flat 0017).

#### Rotation

- Static credentials: rotated on incident + on schedule (cadence per `security/0003-hardening §Patch cadence`)
- Dynamic secrets (database credentials, AWS/Azure STS, etc.): Vault issues short-lived credentials at runtime — no manual rotation
- Rotation workflows live in `ansible-platform/roles/vault-rotate` (future) and are triggered by GitLab CI scheduled pipelines

### Vaultwarden — human password manager

Vaultwarden (Bitwarden-compatible, self-hosted, OSS) stores **human-facing credentials** that operators need to remember but shouldn't type into 15 different browser windows.

#### Scope

| In scope | Out of scope |
|---|---|
| Web app login passwords (Grafana, NetBox, Proxmox, etc. admin UIs) | Any machine-readable credential (goes to Vault) |
| Personal TOTP backup codes | Service account credentials (Vault) |
| Operator's personal notes tied to a credential | SSH keys / passphrases for svc accounts (Vault) |
| Shared org vault for break-glass UI access | Automation-consumed secrets (Vault) |

#### Organization

- **Personal vaults** — one per human operator, encrypted with the operator's own master password
- **Org shared vault** — `by-systems` org, members: all active engineers. Holds credentials that multiple humans may need in emergencies (e.g. break-glass admin UI passwords)
- **No env separation inside Vaultwarden** — humans think in terms of "the GitLab prod UI password", not `secret/prod/gitlab/admin-password`. The env is in the item name if needed.

#### Rules

1. Operators do **not** memorize web app passwords beyond their primary Authentik password — generate random 24+ char passwords in Vaultwarden, copy-paste on login
2. Master password for Vaultwarden is memorized + stored **nowhere else** — if forgotten, the vault is unrecoverable (design)
3. 2FA on Vaultwarden is mandatory — TOTP via Authentik
4. Break-glass: a printed recovery kit is held in a physical safe for the org-shared vault only, containing the recovery key. Never printed for personal vaults.
5. Vaultwarden deployment details (backup, HA, network exposure) are defined in a future `services/` ADR (number assigned when written — `services/0006` is taken by firewall-services). Vaultwarden itself is deployed (`lxc-vaultwarden-01`).

### Redaction

Any time a secret appears in committed text (docs, issues, PR descriptions, commit messages, CHANGELOG, chat logs), it must be redacted:

```
<REDACTED:{type}>
```

**Allowed types:** `password`, `token`, `ssh-pubkey`, `ssh-privkey`, `secret`, `ip-range`, `cert`, `key`

**Rules:**
- Redaction applies to **everything committed or shared** — git, Slack, Discord, email, issues, PR bodies, chat
- **Does not apply** to secret store files themselves — the values must be real for the file to be useful
- Pre-commit hooks run `detect-secrets` on every repo — violations block the commit
- See `identity/0004-os-accounts §11` for the parallel "never cat / echo secret files to terminal" rule

### Separation from library internals

This ADR is **platform-scoped only**. How individual libraries (`lib-opnsense`, `lib-synology-dsm`) accept credentials at their API boundary is governed by each library's own ADRs (typically a `CredentialProvider` abstraction — see `lib/python/0001-design-standard` when refactored from flat 0029). This ADR tells you **where credentials live on the platform**, not how a library receives them.

## Operating model — fabric working copy + Vault mirror (current), AppRole reads (target)

HashiCorp Vault **is deployed** (raft storage, HTTPS, daily raft-snapshot DR). The operating model has two stages:

**Current (fabric + mirror):**

- The controller holds per-credential JSON **fabric files** (`workspace/infra/secrets/fabric/`, mode 0600) — the **deploy-time working copy**: Ansible roles generate-if-absent and read these at provisioning time, so deployments work even while Vault is sealed or being restored.
- `ansible-platform/playbooks/secrets-to-vault.yml` mirrors every fabric file into Vault KV v2 at the §path convention (write-if-absent, drift-reported); `secrets-validate.yml` authenticates each mirrored secret against its **live** service. **Vault is the authoritative mirror and the DR source** — the fabric copy is rebuildable from Vault, and vice versa.
- **Exception that must stay a file:** Vault's own bootstrap (unseal material) can never live only inside Vault; it is held off-platform per §Rules (cold storage), never in the running cluster.

**Target (AppRole runtime reads):** services and rotation workflows read from Vault directly via AppRole + per-service policies (§Per-service access policies); fabric files then shrink to bootstrap-only. Moving a consumer from fabric-read to Vault-read is a per-service change tracked in the roadmap — until then the fabric copy is the contract for deploy-time reads.

## Consequences

- **Two tools, clear boundary** — humans never read Vault, machines never read Vaultwarden. Mix-ups are violations, not accidents.
- **Env isolation is structural** — Vault paths encode env, per-service policies enforce it, and the same policy cannot span envs.
- **Vault HA is non-optional** — if Vault is down, nothing can authenticate, nothing can rotate, nothing can be provisioned. 2-instance active/active deployment is required before production cut-over.
- **Redaction is enforced by hook, not by convention** — `detect-secrets` blocks commits that leak.
- **The transitional JSON state is explicitly non-architecture** — runbook territory, will be deleted after Vault deployment.
- **Vaultwarden and HashiCorp Vault overlap zero** — no secret exists in both tools. If it's in one, it's not in the other.

## Revision triggers

Revise this ADR when:
- A third storage tool is introduced (e.g. 1Password Business, AWS Secrets Manager for cloud workloads)
- Vaultwarden is replaced with a different human password manager
- Vault replaces KV v2 with a different engine for platform secrets (cubbyhole, Transit, etc.)
- The `secret/{env}/{service}/{key}` path structure is insufficient for a new use case (e.g. multi-tenant deployments)
- `detect-secrets` is replaced or extended with a different pre-commit scanner

## CISO mapping

| Framework | Controls covered |
|---|---|
| ISO 27001:2022 | A.5.17 (authentication information), A.8.2 (privileged access rights), A.8.12 (prevention of data leakage), A.8.15 (logging — Vault audit), A.8.24 (use of cryptography — Vault KV v2 at rest) |
| NIS2 | Art. 21(2)(d) (supply chain security — env-isolated credentials), Art. 21(2)(e) (network and information systems security — detect-secrets + audit logs) |
| GDPR | Art. 32(1)(a) (encryption — Vault KV v2 encrypted at rest) |
