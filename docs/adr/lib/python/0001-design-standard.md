# lib/python/0001 — Python Library Design Standard

**Status:** Draft
**Date:** 2026-05-24 (amends 2026-04-14: adds §1.4 singleton + service base patterns to support OPNsense modules — Dnsmasq, radvd, future ABSENT singletons — that do not fit the CRUD-with-UUIDs shape of `BaseManager`. Supersedes flat ADR-0029 v2 (2026-04-08), which superseded v1 (2026-04-04).)
**Scope:** OOP architecture, class hierarchy, transport, async, match keys, validators, error handling, logging, testing, packaging, and the lib-specific PR contract for Python libraries (`lib-*` repos). Does not define generic git workflow, platform-wide redaction, credential storage, or CI hardening gates — each has its own authoritative ADR.
**Related:** `git/0001-workflow`, `security/0001-secret-storage`, `security/0003-hardening`, `identity/0003-machine-credentials`, `naming/0004-automation`, `infra/0001-platform-stack`

---

## Context

BY-SYSTEMS builds Python libraries (`lib-*` repos) to automate infrastructure products — OPNsense, Synology DSM, and future targets. Each library wraps a vendor API and exposes **idempotent resource management** to Ansible, Terraform, and CI pipelines.

Flat ADR-0029 v1 defined the base patterns. v2 added lessons from the `lib-opnsense` implementation:

- God-class `BaseManager` (707 lines) violated separation of concerns
- Missing field validators caused round-trip waste on bad input
- Single match keys caused silent mismatches on duplicate resources
- Partial `try/except` coverage missed errors on read operations
- No logging contract for observability

This ADR (v3 in spirit — the scoped refactor of v2) keeps the v2 decisions, trims everything that overlaps with already-merged scoped ADRs, and moves generic mechanics (git workflow, platform redaction, hardening gates, credential storage) to cross-references.

## Decision

### 1. OOP architecture — separation of concerns

#### 1.1 Source layout

```
src/{product}/
├── __init__.py              # Public API exports (__all__)
├── client.py                # Transport only — HTTP, auth, session, retry
├── credentials.py           # CredentialProvider protocol + Env/Vault implementations
├── exceptions.py            # ALL exceptions (typed hierarchy) — §5
├── logging.py               # structlog config + redaction delegate
├── _types.py                # Shared type aliases
│
├── core/                    # Cross-cutting concerns — independently reusable
│   ├── identity.py          # IdentityResolver — composite match keys + find_existing
│   ├── diff.py              # DiffEngine — state comparison + vendor normalization
│   ├── redaction.py         # Redactor — delegates to security/0001 redaction format
│   ├── validation.py        # FieldValidator protocol + registry + built-in types
│   ├── endpoint.py          # EndpointResolver — URL construction from config
│   ├── logging_helpers.py   # ManagerLogBuilder — structured log construction + timing
│   ├── base_singleton.py    # BaseSingletonManager — fetch/diff/set for singleton config
│   └── base_service.py      # BaseServiceManager — idempotent daemon state control
│
├── managers/
│   ├── base.py              # BaseManager — CRUD-with-UUIDs orchestrator (≤ 200 lines)
│   ├── protocols.py         # Manager typing.Protocol for DI
│   └── {domain}/{entity}.py # Concrete managers — config only (≤ 40 lines)
│
└── models/
    ├── base.py              # EnsureResult frozen dataclass
    └── {domain}/{entity}.py # One frozen dataclass per manager entity
```

Directory layout follows `naming/0004-automation §4` — hierarchical `managers/{domain}/{entity}.py`, not flat `managers/{domain}_{entity}.py`.

#### 1.2 Rules

- **`core/` components have ZERO imports from `managers/` or `client.py`** — they are independently reusable. You can use `core/diff.py` in a CLI tool with no manager stack.
- **`managers/` depend on `core/` and `client.py`** — never on each other. A manager is a thin orchestrator.
- **`models/` are frozen dataclasses** — one per manager entity. No logic, no API calls, no imports from other packages. Managers return typed models; consumers consume them.
- **`exceptions.py` owns ALL exceptions**, including `FieldValidationError`, `AmbiguousMatchError` — no per-module exception definitions.
- **`client.py` owns transport only** — HTTP, authentication, retry, status-to-exception mapping. **Zero business logic.**
- **Every component** must be importable and usable without instantiating the full stack.
- **Every manager docstring** must include **INPUT** (required/optional fields, parent UUIDs) and **OUTPUT** (`EnsureResult` fields) sections.

#### 1.3 Reusability contract

| Component | Usable without | Example standalone use case |
|---|---|---|
| `core/diff.py` | Everything | Compare two dicts (Ansible diff, CI drift detection) |
| `core/redaction.py` | Everything | Redact fields in any context (logs, API responses, CLI output) |
| `core/validation.py` | Everything | Validate input in CLI tools, Ansible modules, Terraform providers |
| `core/identity.py` | Manager (needs list_fn) | Find resource by composite keys in any list of dicts |
| `core/endpoint.py` | Everything | Generate API URLs for docs, curl scripts, API explorers |
| `core/base_singleton.py` | Manager (needs client) | Drive any `settings/get`+`settings/set` daemon config endpoint |
| `core/base_service.py` | Manager (needs client) | Control any `service/{status,start,stop,restart,reconfigure}` daemon |
| `exceptions.py` | Everything | Catch typed errors in any consumer |
| `models/base.py` | Everything | `EnsureResult` usable in any ensure-style function |

#### 1.4 Three manager base patterns — choose the one that matches the API shape

Vendor APIs expose resources in three distinct shapes. Each shape gets its own base class — do not force a singleton config through `BaseManager`.

| API shape | Endpoints | Identity | Base class |
|---|---|---|---|
| **List of resources** (CRUD with UUIDs) | `search{Entity}`, `get{Entity}/{uuid}`, `add{Entity}`, `set{Entity}/{uuid}`, `del{Entity}/{uuid}` | `_match_keys` resolve a row in the list | **`BaseManager`** |
| **Singleton config document** | `{module}/settings/get`, `{module}/settings/set` (no UUIDs, no list) | The resource IS the module — no key needed | **`BaseSingletonManager`** |
| **Service controller** | `{module}/service/{status,start,stop,restart,reconfigure}` (no CRUD) | Target state, not a row | **`BaseServiceManager`** |

**Choosing rule:** if the API surface has both per-resource CRUD AND global settings, write **two managers** — one `BaseManager` subclass per entity collection (e.g. `DnsmasqHostManager`, `DnsmasqRangeManager`), one `BaseSingletonManager` for the module-wide config (e.g. `DnsmasqSettingsManager`), and one `BaseServiceManager` for the daemon (e.g. `DnsmasqServiceManager`). Do not collapse them.

**`BaseSingletonManager` contract:**

- `get() → dict` — fetch current settings (inner payload, unwrapped from `_payload_key`).
- `set(params, check_mode=False) → EnsureResult` — fetch → diff → POST only if drifted → reconfigure (when `_apply_endpoint` is set).
- `ensure(state="present", params, check_mode=False) → EnsureResult` — only `state='present'` is supported; `state='absent'` raises `ValueError` (a singleton config cannot be deleted, only updated).
- Validators (`_validators`), redaction (`REDACT_FIELDS`), and the §8 logging contract apply unchanged.

**`BaseServiceManager` contract:**

- `status() → str` — current daemon status (`'running'` | `'stopped'` | `'disabled'` | `'unknown'`).
- `start() / stop() / restart() / reconfigure()` — direct action methods (NOT idempotent on their own).
- `ensure(state, check_mode=False) → EnsureResult` with valid states:
  - `'running'`      — start if not in `{running}`, noop otherwise.
  - `'stopped'`      — stop if not in `{stopped, disabled, unknown}`, noop otherwise.
  - `'reconfigured'` — **always** acts (no noop semantics — caller is explicitly asking for a config re-read).
- Severity per action: DEBUG=noop, INFO=start/restart/reconfigure, WARNING=stop, ERROR=failure.

Both bases honor the §3 transport contract, §6 validation, §7 exceptions, §8 logging contract, and §10.2 mandatory test set unchanged.

### 2. Dependency injection

All configuration is supplied via constructor parameters. Nothing is hardcoded.

**Client constructor:**

| Parameter | Purpose | Default |
|---|---|---|
| `host` | Target hostname or IP | required |
| `port` | Target port | `443` |
| `verify_ssl` | TLS certificate verification | `True` (override per-deployment) |
| `timeout` | Per-request HTTP timeout (seconds) | `30` |
| `max_retries` | Max retry attempts on transient failure | `3` |
| `retry_backoff` | Backoff factor between retries | `2.0` |
| `credential_provider` | Credential supplier (DI) | required |

**Per-call timeout override:** every client method that makes an HTTP call MUST accept an optional `timeout` parameter to override the global default for a specific call (e.g. firewall reload, cert regeneration).

```python
client = OpnsenseClient(host="vm-opns-01.by-research.be", timeout=30)
await client.reconfigure("firewall/filter/apply", timeout=120)  # slow op, longer timeout
```

**Credential provider pattern:** credentials are never stored in client constructors. A `CredentialProvider` abstraction supplies them at runtime:

```python
class CredentialProvider(Protocol):
    def get_api_key(self) -> str: ...
    def get_api_secret(self) -> str: ...
```

Implementations:
- `EnvCredentialProvider` — reads from environment variables (dev, CI)
- `VaultCredentialProvider` — reads from HashiCorp Vault (AppRole or token auth)

Credential storage paths, rotation, and Vault policies are owned by `security/0001-secret-storage`. This ADR defines the **shape** of the provider, not where credentials live.

### 3. Transport error handling

The client handles ALL transport concerns — HTTP, retry, timeout, status-to-exception mapping.

```python
class ProductClient:
    async def _request(self, method, endpoint, data=None, timeout=None):
        effective_timeout = timeout or self._timeout
        t0 = time.monotonic()
        try:
            # HTTP call with retry logic for transient failures
            result = await self._http.request(...)
            logger.debug("request ok", method=method, endpoint=endpoint,
                         status_code=result.status_code, duration_ms=...)
            return result.body
        except ProductError:
            logger.error("request failed", ...)
            raise
```

**Status code mapping:**

| HTTP status | Exception | Retryable |
|---|---|---|
| 200 + `result=failed` | `ProductValidationError` | No |
| 400 | `ProductValidationError` | No |
| 401 | `ProductAuthError` | No |
| 403 | `ProductPermissionError` | No |
| 404 | `ProductEndpointMissingError` | No |
| 500 | `ProductServerError` | Yes (N retries) |
| Timeout | `ProductTimeoutError` | Yes (N retries) |
| Connection refused | `ProductConnectionError` | Yes (N retries) |

### 4. Async / await

All `lib-*` libraries use Python `asyncio` with `async` / `await`:

- **`httpx.AsyncClient`** for async HTTP
- **`asyncio.gather()`** for parallel operations
- **`async with`** context managers for resource cleanup
- **No `time.sleep()`** — use `asyncio.sleep()` for polling

**Exception:** libraries with a zero-dependency constraint (e.g. `lib-synology-dsm` uses stdlib `urllib` to avoid pulling `httpx`) MAY use synchronous transport internally but MUST still expose an async interface at the public boundary.

### 5. Composite match keys — identity resolution

This section defines the **general pattern** for idempotent ensure-style resource management when the vendor API does not provide a natural unique key. It is the most critical pattern in the library design because it is the foundation of idempotence.

**When match keys matter:** most vendor APIs generate opaque UUIDs on resource creation. The server-side ID is stable but not meaningful — you cannot reconstruct it from the resource's functional content. To determine whether a resource already exists, the library searches a list of existing resources for one whose **functional identity** matches the desired state.

Two shapes of match keys:

- **Single-key** — the resource has a single natural unique field (e.g. Synology share `share_name`). `_match_keys = ["share_name"]` — a degenerate use of the composite pattern.
- **Composite** — the resource has no single natural key; identity is constructed from a tuple of functional fields (e.g. OPNsense firewall filter rule = `(source, destination, port, protocol, action)`). `_match_keys = ["source", "destination", "port", "protocol", "action"]`.

The pattern is the same in both cases.

#### 5.1 Match key contract

Every manager declares `_match_keys: list[str]` — the ordered list of fields that form the resource identity.

| Number of matches from search | Action |
|---|---|
| 0 | Create |
| 1 | Diff → patch if drifted |
| ≥ 2 | Raise `AmbiguousMatchError` — **never silently pick** |

#### 5.2 `AmbiguousMatchError`

```python
class AmbiguousMatchError(ProductError):
    match_keys: dict[str, str]  # The composite key values that matched
    uuids: list[str]            # All UUIDs that matched
```

Logged at ERROR before raising. Consumer uses `uuid=` escape hatch on the next `ensure()` call to target a specific resource explicitly.

#### 5.3 `ensure()` signature

```python
async def ensure(
    self,
    state: str,              # "present" or "absent"
    params: dict[str, Any],  # MUST include ALL match key fields
    check_mode: bool = False,
    uuid: str | None = None, # Escape hatch — bypasses _find_existing
) -> EnsureResult:
```

#### 5.4 Description is NOT identity

For resources without API-enforced unique names (firewall rules, NAT rules), the `description` field is a **managed field**, not a match key. An admin may rename the resource in the vendor UI — the next `ensure()` run detects the drift and corrects it back. **The playbook is the source of truth**, not the current UI state.

### 6. Field validators

#### 6.1 Mandatory per manager

Every manager declares `_validators: dict[str, dict]` mapping field names to validation specs. Validation runs in `ensure(state="present")` **before any API call** — invalid input is rejected locally with a typed error instead of causing a round-trip to the vendor API.

#### 6.2 Built-in validator types

Derived from vendor MVC model field types (example references: OPNsense MVC field types, Synology DSM schema):

| Type | Validation | Reference field type |
|---|---|---|
| `str` | `max_length`, `regex` pattern | TextField |
| `enum` | Allowed-values list | OptionField |
| `int` | `min`, `max` range | IntegerField |
| `bool_str` | `"0"` or `"1"` only | BooleanField |
| `email` | Email regex | EmailField |
| `ip` | IPv4 / IPv6 via stdlib `ipaddress` | NetworkField |
| `cidr` | Network + prefix via stdlib `ipaddress` | NetworkField |
| `mac` | MAC address format | MacAddressField |
| `port` | 1–65535 or range | PortField |
| `hostname` | RFC 952 / 1123 | HostnameField |

#### 6.3 Extensible registry

Validators implement a `FieldValidator` protocol. Custom validators can be registered via the registry without modifying library source — consumers with exotic constraints register their own.

#### 6.4 Documentation requirement

Each manager's docstring MUST link to the vendor API documentation and the MVC model field-type definitions that the validators are derived from. When the vendor updates its schema, the linked doc is the source of truth for what the validators should enforce.

### 7. Typed exception hierarchy

Every library defines a product-scoped base exception with typed subclasses:

```
ProductError (base)
├── ProductAuthError (401)
├── ProductPermissionError (403)
├── ProductEndpointMissingError (404)
├── ProductValidationError (400 + result=failed)
├── ProductServerError (500)
├── ProductTimeoutError
├── ProductConnectionError
├── AmbiguousMatchError (multiple resources match composite keys)
└── FieldValidationError (client-side param validation failure)
```

#### 7.1 `try` / `except` / log / raise — on ALL methods that can throw

**Not just mutations.** Every method that makes an API call, or calls code that can fail, wraps its body in `try/except`:

```python
# Manager methods that need try/except:
async def list(...)         # API call
async def get(...)          # API call
async def get_schema(...)   # API call
async def create(...)       # API call + apply
async def update(...)       # API call + apply
async def delete(...)       # API call + apply
async def ensure(...)       # validation + orchestration
async def _apply(...)       # API call
```

Pattern:

```python
try:
    result = await self._client.method(endpoint)
except Exception as exc:
    logger.error("action failed: %s", exc, extra={...})
    raise
```

**Pure logic methods** (`_compute_diff`, `_redact`, `_match_label`) do **NOT** need `try/except` — if they throw, it's a bug; let it crash.

#### 7.2 Consumer contract

Consumers MUST use `try/except/finally`:

```python
try:
    result = await mgr.ensure("present", params)
except FieldValidationError as exc:
    # Client-side validation — bad params
except AmbiguousMatchError as exc:
    # Multiple resources match — use uuid= or deduplicate
except ProductValidationError as exc:
    # Server-side validation — API rejected params
except ProductAuthError:
    # Auth failed — do not retry
except ProductTimeoutError:
    # Timeout — may retry
except ProductError as exc:
    # Generic fallback
finally:
    # ALWAYS runs — cleanup, close connections, report status
```

### 8. Logging contract

#### 8.1 Manager logging

Every `ensure()` outcome logs these fields:

| Log site | match_keys | uuid | before | after | changed | duration_ms |
|---|---|---|---|---|---|---|
| create (incl. check_mode) | yes | yes* | — | yes | yes | yes |
| update (incl. check_mode) | yes | yes | yes | yes | yes | yes |
| delete (incl. check_mode) | yes | yes | yes | — | yes | yes |
| noop | yes | yes | — | — | yes | yes |
| error | yes | yes* | yes* | — | — | yes |
| ambiguous_match | yes | all | — | — | — | — |
| validation_failed | — | — | — | — | — | — |

**Severity:** DEBUG = noop, INFO = create/update, WARNING = delete, ERROR = failure / ambiguity / validation.

`duration_ms` uses `time.monotonic()` — zero overhead, no sleeps.

#### 8.2 Client logging

Every HTTP call:

| Outcome | Level | Fields |
|---|---|---|
| Success (2xx) | DEBUG | `method`, `endpoint`, `status_code`, `duration_ms` |
| Retry (500 / network) | WARNING | `method`, `endpoint`, `status_code`, `attempt`, `duration_ms` |
| Final failure | ERROR | `method`, `endpoint`, `status_code`, `duration_ms` |

#### 8.3 Structured logging

Use **`structlog`** with:
- Colorized console output for development
- JSON file output for Loki / Promtail ingestion (per `infra/0006-logging`)
- Redaction processor applied before both outputs (delegates to `core/redaction.py`, which implements the `security/0001-secret-storage` redaction format)
- `duration_ms` on every log entry

### 9. Secret redaction — delegated

Libraries redact secrets in logs and in `EnsureResult.before` / `EnsureResult.after` diff fields. **Redaction format is owned by `security/0001-secret-storage`** (the `<REDACTED:{type}>` pattern). This ADR does not reproduce the format.

Each manager declares `REDACT_FIELDS: set[str]` — the set of field names that must be redacted when the manager processes them. The actual redaction is performed by `core/redaction.py`, which delegates format decisions to the platform standard.

**Rules:**
- Passwords, OTP seeds, vault tokens: **always full redaction** — never show any part
- API keys and session tokens: partial reveal allowed per `security/0001-secret-storage` for verification (e.g. first-4 + last-4)
- Redaction runs as a `structlog` processor + in `EnsureResult` before/after fields — both paths go through `core/redaction.py`
- **`core/redaction.py` is the single source of truth** inside a library. `logging.py` and `managers/base.py` both delegate to it.

### 10. Testing

#### 10.1 Test structure — per concern, per manager

Tests mirror the `src/` layout (per `naming/0004-automation §8`):

```
tests/
├── unit/
│   ├── core/
│   │   ├── test_identity.py        # IdentityResolver
│   │   ├── test_diff.py            # DiffEngine
│   │   ├── test_redaction.py       # Redactor
│   │   ├── test_validation.py      # Validators
│   │   ├── test_endpoint.py        # EndpointResolver
│   │   └── test_logging_helpers.py # ManagerLogBuilder
│   ├── managers/
│   │   ├── auth/
│   │   │   ├── test_user.py               # AuthUserManager CRUD
│   │   │   └── test_user_validation.py    # AuthUser field validation
│   │   └── firewall/
│   │       ├── test_alias.py
│   │       └── test_filter_ambiguous.py   # Duplicate detection
│   └── ...
├── integration/
│   ├── auth/
│   │   └── test_user.py               # Auth domain lifecycle
│   └── firewall/
│       ├── test_alias.py
│       └── test_ambiguous.py          # Duplicate detection on live device
└── smoke/
    └── test_smoke.py                  # Import + instantiation only
```

One test file per concern per manager. Small files, clear scope.

#### 10.2 Mandatory error-handling test set

Every manager test suite MUST include these six cases:

1. `create` / `update` / `delete` failure — logs ERROR, re-raises unchanged
2. Exception type preserved through the manager layer
3. `FieldValidationError` raised **before** API call for bad params
4. `AmbiguousMatchError` raised with correct UUIDs on duplicate
5. Consumer `try/except/finally` pattern works correctly
6. Transport errors: 401, 403, 404, 400, 500, timeout, connection

#### 10.3 Quality gates

Library CI enforces the following gates **locally before commit** and again in CI:

| Gate | Tool | Command |
|---|---|---|
| Lint | `ruff` | `ruff check src/ tests/` |
| Format | `ruff` | `ruff format --check src/ tests/` |
| Types | `mypy` | `mypy src/ --ignore-missing-imports` |
| Security (Python-specific) | `bandit` | `bandit -r src/ -q -ll` |
| CVE scan | `pip-audit` | `pip-audit` |
| Unit tests | `pytest` | `pytest tests/unit/ -q` |

**Platform-wide container / image hardening gates** (Trivy, Lynis, CIS benchmarks, patch cadence) are owned by `security/0003-hardening §4 Trivy in CI` and are applied at the repo CI level, independent of the Python library gates above.

**Python version matrix:** 3.10, 3.11, 3.12, 3.13.

### 11. Git workflow — library-specific DoD

**Workflow mechanics** (branching, commits, merge strategy, agent boundary, protected branches) are owned by `git/0001-workflow`. This ADR does not duplicate them.

**Library-specific Definition of Done** — every library PR satisfies all of:

- All quality gates pass (`ruff`, `mypy`, `bandit`, `pip-audit`, `pytest` — per §10.3)
- No manager without a `_validators` declaration
- No manager without **INPUT / OUTPUT** docstring sections (required/optional fields, parent UUIDs, `EnsureResult` fields)
- No validators without unit tests
- No integration test without documented safety boundaries (what the test creates on the live device, how it cleans up, what it must NOT touch)
- CI green before merge — `gh run list` confirms
- `docs/api-coverage.md` updated in the same PR for any new manager (status, match keys, notes, reference URL)
- Integration tests committed — smoke-only is **not** enough for a manager PR
- **Secrets never output in terminal commands during development** — per `security/0001-secret-storage §Redaction` and `identity/0004-os-accounts §11`. Read silently, write to `.env` only.
- Merge strategy: `gh pr merge N --merge --delete-branch` (merge commit, not squash, not rebase) — per `git/0001-workflow`

### 12. Packaging

- **Build system:** `pyproject.toml` with `hatchling`
- **Install method (current):** `pip install git+https://github.com/{org}/{repo}.git@{tag}`
- **Install method (target):** Nexus Repository OSS as the universal proxy and cache for PyPI — see `infra/0001-platform-stack §Tool inventory`. When Nexus is deployed, libraries are published to the internal PyPI index on Nexus, and `pip install` reads from Nexus (which falls through to `pypi.org` for upstream packages and caches them permanently).
- **Versioning:** Conventional Commits + release-please — per `git/0001-workflow §Tagging`
- **Runtime deps:** minimal (`httpx`, `structlog`)
- **Dev deps:** `ruff`, `mypy`, `pytest`, `pytest-cov`, `pytest-asyncio`, `bandit`, `pip-audit`
- **Devcontainer:** `.devcontainer/devcontainer.json` with Python + `ruff` + `mypy` pre-installed for reproducible local dev

Ansible modules and Terraform providers consume the library via the current install method (`pip install git+...@tag`) — never by cloning the git repo and importing from source.

## Consequences

- All `lib-*` repositories MUST follow this standard. No "we'll get to it next refactor."
- **`lib-opnsense`** required a refactor to extract `core/` components (scope-of-work document exists in the repo).
- **`lib-synology-dsm`** requires an alignment audit against this ADR.
- New libraries start compliant from day one.
- The **`core/` module is the point of leverage** — extracting cross-cutting concerns there makes each manager tiny (≤ 40 lines of config) and makes the cross-cutting concerns independently testable.
- **Match keys enable true idempotence** — no silent duplicate creation, no stale resource drift. `AmbiguousMatchError` forces the human to make the duplicate-handling decision explicitly.
- **Field validators catch errors before the vendor round-trip** — saves API quota and surfaces bad inputs with better error messages than the vendor API would.
- **`try/except` on all methods that throw** means consumers always get typed errors, not raw `httpx` exceptions leaking through.
- **Logging contract** makes library behavior observable in Loki without extra instrumentation.
- **Packaging path is clear** — `git+https` today, Nexus tomorrow, no breaking change for consumers (just the index URL).

## Revision triggers

Revise this ADR when:
- A new domain-specific pattern is needed that doesn't fit the current `core/` component set
- A vendor API exposes a resource shape that fits neither `BaseManager` (CRUD-with-UUIDs), `BaseSingletonManager` (settings get/set), nor `BaseServiceManager` (service controller) — add a fourth base
- An additional exception type is needed across all libraries
- The match-key pattern is insufficient for a new resource class (unlikely — composite keys cover every case so far)
- A new testing framework replaces `pytest`
- A new linter or type checker replaces `ruff` / `mypy`
- Nexus is deployed and `pip install` migrates from `git+https` to the internal index
- Python version support drops 3.10 or adds 3.14+
- A new Python library repo is started under `lib-*` naming — confirm it follows this standard on day one
- A peer ADR is created for Ansible collections (`lib/ansible/0001-design-standard`) or TypeScript libraries (`lib/typescript/0001-design-standard`) — cross-reference from here

## CISO mapping

| Framework | Controls covered |
|---|---|
| ISO 27001:2022 | A.8.25 (secure development lifecycle — DoD, testing, validators), A.8.28 (secure coding — typed exceptions, no exceptions leaked raw from transport), A.8.9 (configuration management — versioned ADR controls library structure), A.5.17 (authentication information — credentials via DI provider, never stored in client) |
| NIS2 | Art. 21(2)(e) (security in development and acquisition — library design standard ensures consistent security properties across every automation target) |
