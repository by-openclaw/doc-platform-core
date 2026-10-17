# security/0001 — Secret Storage

**Status:** Draft
**Date:** 2026-04-13 (supersedes flat ADR-0011, 2026-03-31 + flat ADR-0016, 2026-04-02)
**Amended:** 2026-10-11 (approved by the owner) — people read in Vault what is meant for them (§People reading in Vault); the controller keeps only what opens Vault (§Operating model)
**Amendment proposed:** 2026-10-17 — document classes, mandatory purpose metadata, the generated register (§Document classes, purpose and register). Not in force until the owner approves this PR.
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
| **HashiCorp Vault** | Programmatic machine secret store | API tokens, SSH passphrases, service account credentials, `ansible_become_pass`, OIDC client secrets, database credentials, TLS private keys, webhook HMAC keys | Ansible, CI/CD, applications, service accounts — and, read-only, the people a document is meant for (§People reading in Vault) |
| **Vaultwarden** | Human password manager | What a person creates and keeps: app passwords of mail programs, recovery and TOTP backup codes, logins of outside portals (suppliers, registrars) | Humans via browser extension / mobile app |

**The boundary is strict:**

- Machine processes read from **Vault** via API (AppRole / token) — never from Vaultwarden
- A login that automation creates and uses — a service's local administrator, an emergency account — lives in **Vault** only. The people entitled to it read it there (§People reading in Vault); it is never copied into Vaultwarden
- What a person creates for themselves lives in **Vaultwarden** — never in Vault. Nothing is written into Vault by hand
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

#### Document classes, purpose and register (proposed 2026-10-17)

The path convention above says where a document sits; it says nothing about what the document
is **for**, who **presents** it, who **reads** it and how it **rotates**. On 2026-10-17 Vault held
205 documents under the tiers; 7 carried purpose metadata, the rest an ad hoc `note`, `_note` or
`purpose` field, or nothing — the owner could not tell what `users/<person>/firewall` or `pve/oidc`
were for. This section closes that gap with one rule, four classes, mandatory metadata, and a
register generated from Vault itself.

**One rule: a credential lives under the party that presents it.**

| Class | Who presents it | Path | Examples |
|---|---|---|---|
| **system** | the service itself, to run or to authenticate as a client | `{tier}/{service}/{key}`; per node `{tier}/{service}/{hostname}/{key}` | `prod/proxmox/oidc` (Proxmox presents it to Authentik), `prod/proxmox/<node>/root`, `dev/windows/<host>/administrator`, `prod/gitlab/db-pgsql` |
| **identity** | a person or a machine account, to a system | `{tier}/users/{person}/{key}`, `{tier}/svc-{function}/{key}` | `prod/users/adm_yboujraf/keys`, `prod/svc-ansible/gitlab-api`, `dev/svc-rune/proxmox-terraform`, `prod/svc-rune/github-token`. The platform has two machine identities, `svc-ansible` and `svc-rune`; the second agent identity (`svc-opus`) is retired (owner, 2026-10-17). |
| **break-glass** | a human at a console when Authentik and Vault are down | `{tier}/break-glass/{key}` | `prod/break-glass/os`, `prod/break-glass/opnsense`, `prod/break-glass/ssh-key` |
| **placeholder** | nobody — a value a product forces into a record whose authentication is elsewhere | avoided; when unavoidable, class `placeholder`, no reader | the local GUI password OPNsense requires on an LDAP-authenticated user |

Rules that follow:

1. **Tier = the tier of the party.** A hypervisor's documents sit under the node's tier (`dev/proxmox/…` for the dev node, `prod/proxmox/…` for the live node — `poc` is not a tier, `naming/0001 §4.1`). Guests of another tier that use that node's token read it through policy (`platform_proxmox_env`), never through a path of their own tier.
2. **A hostname appears only in the per-node segment** of a system document, never in the service slot (`pve.dev` was wrong). Service names are the service short codes (`proxmox`, `opnsense`, `windows`), one per product.
3. **Tokens, passwords and keys an identity presents to a system live under the identity**, named after the system: `svc-rune/proxmox-admin`, `svc-rune/proxmox-terraform`, `svc-ansible/jumpserver-api`. A system never holds another party's credential (the live node's `proxmox/admin` bundling three identities' tokens is the counter-example).
4. **A key belongs to whoever holds the private half.** People: `users/<person>/keys`. Machine accounts: `svc-<function>/ssh-key` + `ssh-passphrase`. There is no "key of a system": a system *accepts* identity keys. Transitional per-system key documents (`proxmox/ssh-key`, `windows/ssh-key`, `opnsense/ssh-key`, 2026-10-16) retire once the systems accept the identity key.
5. **Mandatory custom metadata on every document** (KV v2 metadata, never in the data): `purpose` (one sentence), `class` (system | identity | break-glass | placeholder), `presenter`, `writer` (the role or play that creates and rotates it), `readers` (roles, plays, people groups), `rotation` (`<days>` or `on-incident` or `static`). The `note` / `_note` / `purpose` **data** fields migrate into metadata and disappear from the data.
6. **The register is generated, never hand-written.** `playbooks/vault-register.yml` lists every document under the tiers, fails on a missing metadata field or on a document with no reader (unless class `break-glass` or `placeholder`), and writes `docs/vault-register.md` in ansible-platform (one row per document: path, class, purpose, presenter, writer, readers, rotation, created, last rotated). It is the asset inventory of secrets (ISO 27001 A.5.9) and runs in CI like `docs/register.md`.
7. **An identity exists once; tiers do not copy it.** A person or a machine account is one identity, kept in the tier of the identity provider (`prod/users/<person>/…`, `prod/svc-<function>/…`): one key pair, one sign-in. What that identity presents to a system of another tier sits under that tier, under the identity (`dev/svc-rune/proxmox-terraform`, `test/users/<person>/firewall`). Reach is widened by **groups** (Authentik groups → Vault policies, service ACLs), never by a second copy of the identity's material in another tier — so a group grant is revoked in one place and a person's key is one key to rotate.
8. **Retire what the rule exposes:** placeholders a product can scramble instead of store, one-off records nobody signs in with that are not break-glass.

**Migration (copy → switch the readers → retire, `roles/vault_tidy`, one PR per line, no secret regenerated):**

| Today | Target | Class |
|---|---|---|
| `prod/pve.dev/root` | `dev/proxmox/br-pmox-03/root` | system, per node |
| `prod/pve/oidc` | `prod/proxmox/oidc` | system |
| `prod/proxmox/root` | `prod/proxmox/srv-proxmox-poc-01/root` | system, per node |
| `prod/proxmox/admin` (admin + terraform token pairs; the audit pair of the retired agent is removed) | `prod/svc-rune/proxmox-admin`, `prod/svc-rune/proxmox-terraform` | identity |
| `dev/proxmox/tokens/{svc-rune-admin,svc-rune-terraform}` (the retired agent's token is removed) | `dev/svc-rune/proxmox-admin`, `dev/svc-rune/proxmox-terraform` | identity |
| `prod/ci/github-rune`, `prod/ci/git-identity` (`ci/github-opus` is retired with its agent) | `prod/svc-rune/github-token`, `prod/svc-rune/git-identity` | identity |
| `prod/{proxmox,windows,opnsense}/ssh-key` + `ssh-passphrase` | retired after the acceptors take `svc-ansible-prod`'s key | — |
| `prod/resend/by-research` | **kept** — the owner decided on 2026-10-11 that the external relay must keep working though it is not used now (`platform_vault_restored_documents`); class `system`, metadata says so | system |
| `prod/users/<person>/firewall` | scrambled on the firewall if the product allows; else kept as class `placeholder` | placeholder |

Firewalls already fit (`prod/opnsense/*`, `test/opnsense/*`); a per-node segment appears only when a tier has two.

**Compliance mapping of this section** — ISO 27001:2022 A.5.9 (inventory: the generated register), A.5.16 / A.5.17 (one credential per identity, no shared or unused authentication information, placeholders named), A.5.18 / A.8.2 (per-service policies, people's own area, break-glass apart), A.8.5 (people authenticate through Authentik only), A.8.24 (keys under their identity, escrowed, dated, 365-day rule, passphrase state recorded); NIS2 Art. 21(2)(h) cryptography, (i) access control and asset management, (j) MFA; ISO 27002 guidance on 5.17 for the lifecycle of authentication information.

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

#### People reading in Vault

Automation creates logins a person sometimes has to type: the local administrator of a service that single sign-on does not cover, an emergency account, a first password. They are machine secrets — roles read them at every run — so they stay in Vault, in one place. The person reads them there.

1. **Sign-in through the identity provider only** (OIDC, same second factor as everywhere). Vault has no local user for a person.
2. **By group.** The administrators' group of a service reads that service's login document — the path is declared per service in the catalog of `ansible-platform`. A document that mixes a person's login with machine-only values (a database password, an application key) is split first: the group is given a login, never a service's whole secret set.
3. **Per person.** `secret/{env}/users/{username}/…` is readable by that person and nobody else. What automation generates for one person (a first sign-in password, a personal firewall password, personal key material) is handed over there; once the person has taken it into their own password manager it is removed from the area.
4. **Read-only.** A person writes nothing into Vault; automation is the only writer.
5. **Administrators of Vault** manage it and are denied the other people's areas by policy. An administrator can change a policy: this is **detected, not prevented**. Every read and every policy change is in the audit log, shipped to the log store, with an alert on a read in a person's area by anyone else, on a policy change outside an automation run, and on any use of a root token.
6. **When the identity provider is down** people do not reach Vault. Getting back in is the emergency kit of `identity/0004-os-accounts`, not Vault's UI.

Why not copy these logins into Vaultwarden: its items are encrypted by the client, its official tool cannot create its own account, so a bridge would need a master password set by hand at every installation — and the same secret would then live in both tools, to be rotated in both.

### Vaultwarden — human password manager

Vaultwarden (Bitwarden-compatible, self-hosted, OSS) stores **human-facing credentials** that operators need to remember but shouldn't type into 15 different browser windows.

#### Scope

| In scope | Out of scope |
|---|---|
| Logins a person created: outside portals, tools without single sign-on that automation does not manage | Any credential automation creates or reads (Vault) — a service's local administrator included |
| Personal TOTP backup codes | Service account credentials (Vault) |
| Operator's personal notes tied to a credential | SSH keys / passphrases for svc accounts (Vault) |
| Shared collections for logins typed by people and used by several (a supplier portal) | The platform's emergency logins (Vault, read by group; the kit of `identity/0004-os-accounts`) |

#### Organization

- **Personal vaults** — one per human operator, encrypted with the operator's own master password
- **Org shared collections** — `by-systems` org, members: all active engineers. Hold logins that people created and several of them use (a supplier portal). The platform's emergency logins are not here: they stay in Vault (§People reading in Vault)
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

## Operating model — Vault is the store; the controller keeps only what opens it

HashiCorp Vault **is deployed** (raft storage, HTTPS, daily raft-snapshot DR).

- **Roles create a credential in Vault and read it from Vault.** A role that needs a secret gets it or creates it there (`roles/vault_secret`); no role writes a credential file.
- **The controller keeps only the files a play has to read before Vault can answer:** Vault's unseal material and logins, the automation account's password, what a firewall is built from, the emergency kit, Vault's snapshots — and, until they are moved, the few files that tools outside `ansible-platform` still read. Each is declared with its reader in `ansible-platform/inventories/prod/group_vars/all/controller_files.yml`; `playbooks/controller-files.yml` fails on any other file of the folder.
- `secrets-validate.yml` authenticates Vault's values against the **live** services.
- **Exception that must stay a file:** Vault's own bootstrap (unseal material) can never live only inside Vault; it is held off-platform per §Rules (cold storage), never in the running cluster.

Until 2026-10-11 the controller held one JSON file per credential (the *fabric* files), mirrored into Vault by `secrets-to-vault.yml`. That stage is over: every file was compared with Vault, the files without a reader were removed, and that playbook no longer writes back a document that was archived.

**Target (AppRole runtime reads):** services and rotation workflows read from Vault directly via AppRole + per-service policies (§Per-service access policies). Moving a consumer to a runtime read is a per-service change tracked in the roadmap.

## Consequences

- **Two tools, clear boundary** — machines never read Vaultwarden; people read in Vault only what their group or their own area gives them, read-only and logged, and write nothing there. Mix-ups are violations, not accidents.
- **Env isolation is structural** — Vault paths encode env, per-service policies enforce it, and the same policy cannot span envs.
- **Vault HA is non-optional** — if Vault is down, nothing can authenticate, nothing can rotate, nothing can be provisioned. 2-instance active/active deployment is required before production cut-over.
- **Redaction is enforced by hook, not by convention** — `detect-secrets` blocks commits that leak.
- **The JSON files of the first months are gone** (2026-10-11) — what remains on the controller is declared and checked (§Operating model).
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
| ISO 27001:2022 | A.5.17 (authentication information), A.5.18 (access rights — people's read rights by group and per person), A.8.2 (privileged access rights), A.8.12 (prevention of data leakage), A.8.15 (logging — Vault audit), A.8.24 (use of cryptography — Vault KV v2 at rest) |
| NIS2 | Art. 21(2)(d) (supply chain security — env-isolated credentials), Art. 21(2)(e) (network and information systems security — detect-secrets + audit logs) |
| GDPR | Art. 32(1)(a) (encryption — Vault KV v2 encrypted at rest) |
