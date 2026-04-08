# ADR-0029 — Python Library Design Standard (v2)

**Date:** 2026-04-08
**Status:** Accepted (supersedes v1 dated 2026-04-04)
**Deciders:** Youssef Boujraf
**Tags:** `python`, `lib`, `design-standard`, `ensure`, `asyncio`, `oop`, `separation-of-concerns`

---

## 1. Status

Accepted — supersedes ADR-0029 v1 (archived as `archive/0029-python-library-design-standard.v1.md`).

**Changes from v1:**
- Separation of concerns: core/ module with independently reusable components
- Composite match keys with AmbiguousMatchError
- Field validators mandatory per manager (from OPNsense MVC model definitions)
- try/except/log/raise on ALL methods that can throw (not just mutations)
- Transport: global defaults overridable per-call
- Git flow: no PR without CI green, use templates for everything
- Logging contract with duration_ms on every call

---

## 2. Context

BY-SYSTEMS builds Python libraries (`lib-*` repos) to automate infrastructure products — Synology DSM, OPNsense, and future targets. Each library wraps a vendor API and exposes idempotent resource management to Ansible, Terraform, and CI pipelines.

v1 defined the base patterns. v2 adds lessons from lib-opnsense implementation:
- God class BaseManager (707 lines) violated separation of concerns
- Missing field validators caused round-trip waste on bad input
- Single match keys caused silent mismatches on duplicate resources
- Partial try/except coverage missed errors on read operations
- No logging contract for observability

---

## 3. Decision — OOP Architecture + Separation of Concerns

### 3.1 Source layout

```
src/{product}/
├── __init__.py              # Public API exports (__all__)
├── client.py                # Transport only (HTTP, auth, session)
├── credentials.py           # Credential providers (DI)
├── exceptions.py            # ALL exceptions (typed hierarchy)
├── logging.py               # Structlog config (delegates to core/redaction)
├── _types.py                # Shared type aliases
│
├── core/                    # Cross-cutting concerns — independently reusable
│   ├── identity.py          # IdentityResolver — composite match keys + find_existing
│   ├── diff.py              # DiffEngine — state comparison + vendor normalization
│   ├── redaction.py         # Redactor — unified field redaction
│   ├── validation.py        # FieldValidator protocol + registry + built-in types
│   ├── endpoint.py          # EndpointResolver — URL construction from config
│   └── logging_helpers.py   # ManagerLogBuilder — structured log construction + timing
│
├── managers/
│   ├── base.py              # BaseManager — thin orchestrator (≤200 lines)
│   ├── protocols.py         # Manager typing.Protocol for DI
│   ├── {domain}_{entity}.py # Concrete managers — config only (≤40 lines)
│   └── ...
│
└── models/
    ├── base.py              # EnsureResult frozen dataclass
    ├── auth_user.py         # AuthUser — one model per manager
    ├── auth_group.py        # AuthGroup
    ├── fw_alias.py          # FwAlias
    ├── fw_filter.py         # FwFilterRule
    └── ...                  # One frozen dataclass per manager entity
```

### 3.2 Rules

- **core/** components have ZERO imports from managers/ or client.py — they are independently reusable
- **managers/** depend on core/ and client.py — never on each other
- **models/** are frozen dataclasses — one per manager entity. No logic, no API calls, no imports from other packages. Managers return typed models, consumers consume them
- **exceptions.py** owns ALL exceptions including FieldValidationError, AmbiguousMatchError
- **client.py** owns transport, authentication, retry, status-to-exception mapping — zero business logic
- **Every component** must be importable and usable without instantiating the full stack

### 3.3 Reusability contract

| Component | Usable without | Example standalone use case |
|---|---|---|
| core/diff.py | Everything | Compare any two dicts (Ansible diff, CI drift detection) |
| core/redaction.py | Everything | Redact fields in any context (logs, API responses, CLI output) |
| core/validation.py | Everything | Validate input in CLI tools, Ansible modules, Terraform providers |
| core/identity.py | Manager (needs list_fn) | Find resource by composite keys in any list of dicts |
| core/endpoint.py | Everything | Generate API URLs for docs, curl scripts, API explorers |
| exceptions.py | Everything | Catch typed errors in any consumer |
| models/base.py | Everything | EnsureResult usable in any ensure-style function |

---

## 4. Decision — Dependency Injection

All configuration is supplied via constructor parameters. Nothing is hardcoded.

**Client constructor parameters:**

| Parameter | Purpose | Default |
|---|---|---|
| `host` | Target hostname or IP | required |
| `port` | Target port | 443 |
| `verify_ssl` | TLS certificate verification | False |
| `timeout` | Per-request HTTP timeout (seconds) | 30 |
| `max_retries` | Maximum retry attempts on transient failure | 3 |
| `retry_backoff` | Backoff factor between retries (seconds) | 2.0 |

**Per-call override:** Every client method that makes an HTTP call MUST accept optional
`timeout` parameter to override the global default for that specific call.

```python
# Global default: 30s
client = OpnsenseClient(host="...", timeout=30)

# Override for slow operation: 120s
await client.reconfigure("firewall/filter/apply", timeout=120)
```

**Credential provider pattern:** Credentials are never stored in client constructors. A credential provider abstraction supplies them at runtime:

- `EnvCredentialProvider` — reads from environment variables
- `VaultCredentialProvider` — reads from HashiCorp Vault (AppRole or token auth)

---

## 5. Decision — Transport Error Handling

### 5.1 Client transport layer

The client handles ALL transport concerns: HTTP, retry, timeout, status-to-exception mapping.

```python
class ProductClient:
    async def _request(self, method, endpoint, data=None, timeout=None):
        """Every HTTP call goes through this method."""
        effective_timeout = timeout or self._timeout
        t0 = time.monotonic()
        try:
            # ... HTTP call with retry logic ...
            logger.debug(...)  # success
            return body
        except ProductError:
            logger.error(...)  # all errors logged at client level
            raise
```

**Status code mapping:**

| HTTP Status | Exception | Retryable? |
|---|---|---|
| 200 + result=failed | ProductValidationError | No |
| 400 | ProductValidationError | No |
| 401 | ProductAuthError | No |
| 403 | ProductPermissionError | No |
| 404 | ProductEndpointMissingError | No |
| 500 | ProductServerError | Yes (retry N times) |
| Timeout | ProductTimeoutError | Yes (retry N times) |
| Connection refused | ProductConnectionError | Yes (retry N times) |

### 5.2 Client logging contract

Every HTTP call is logged:

| Outcome | Level | Fields |
|---|---|---|
| Success (2xx) | DEBUG | method, endpoint, status_code, duration_ms |
| Retry (500/net) | WARNING | method, endpoint, status_code, attempt, duration_ms |
| Final failure | ERROR | method, endpoint, status_code, duration_ms |

---

## 6. Decision — Async / Await

All `lib-*` libraries use Python `asyncio` with `async/await`:

- **`httpx.AsyncClient`** for async HTTP
- **`asyncio.gather()`** for parallel operations
- **`async with` context managers** for resource cleanup
- **No `time.sleep()`** — use `asyncio.sleep()` for polling

**Exception:** Libraries with zero-dependency constraint (e.g. lib-synology-dsm) may use stdlib urllib but must still expose async interfaces.

---

## 7. Decision — Composite Match Keys + Identity Resolution

### 7.1 Match key contract

Every manager declares `_match_keys: list[str]` — the fields that form the resource identity.

| # matches from search | Action |
|---|---|
| 0 | Create |
| 1 | Diff → patch if drifted |
| ≥2 | Raise AmbiguousMatchError — never silently pick |

Legacy `_match_key: str` is supported for backward compatibility (treated as single-element list).

### 7.2 AmbiguousMatchError

```python
class AmbiguousMatchError(ProductError):
    match_keys: dict[str, str]  # The composite key values that matched
    uuids: list[str]            # All UUIDs that matched
```

Logged at ERROR before raising. Consumer uses `uuid=` escape hatch to target a specific resource.

### 7.3 ensure() signature

```python
async def ensure(
    self,
    state: str,              # "present" or "absent"
    params: dict[str, Any],  # Must include ALL match key fields
    check_mode: bool = False,
    uuid: str | None = None, # Escape hatch — bypasses _find_existing
) -> EnsureResult:
```

### 7.4 Description is NOT identity

For resources without API-enforced unique names (firewall rules, NAT rules),
`description` is a managed field, not a match key. Admin can rename in WebGUI —
next `ensure()` corrects it back. The playbook is the source of truth.

---

## 8. Decision — Field Validators

### 8.1 Mandatory per manager

Every manager declares `_validators: dict[str, dict]` mapping field names to validation specs.
Validation runs in `ensure(state="present")` BEFORE any API call.

### 8.2 Validator types (from vendor MVC model definitions)

| Type | Validation | Reference |
|---|---|---|
| str | max_length, regex pattern | TextField |
| enum | allowed values list | OptionField |
| int | min, max range | IntegerField |
| bool_str | "0" or "1" only | BooleanField |
| email | email regex | EmailField |
| ip | IPv4/IPv6 via stdlib ipaddress | NetworkField |
| cidr | network/prefix via stdlib ipaddress | NetworkField |
| mac | MAC address format | MacAddressField |
| port | 1-65535 or range | PortField |
| color | 6 hex digits (no # prefix) | TextField + constraint |
| hostname | RFC 952/1123 | HostnameField |

### 8.3 Extensible registry

Validators implement a `FieldValidator` protocol. Custom validators can be registered
via the registry without modifying source code.

### 8.4 Documentation requirement

Each manager's docstring MUST link to the vendor API docs and MVC model field type
definitions that the validators are derived from.

---

## 9. Decision — Error Handling

### 9.1 Typed exception hierarchy

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

### 9.2 try/except/log/raise — on ALL methods that can throw

Not just mutations. **Every method that makes an API call or calls code that can fail:**

```python
# Manager methods — ALL of these need try/except:
async def list(...)         # API call
async def get(...)          # API call
async def get_schema(...)   # API call
async def create(...)       # API call + apply
async def update(...)       # API call + apply
async def delete(...)       # API call + apply
async def ensure(...)       # validation + orchestration
async def _apply(...)       # API call

# Pattern:
try:
    result = await self._client.method(endpoint)
except Exception as exc:
    logger.error("action failed: %s", exc, extra={...})
    raise
```

**Pure logic methods** (_compute_diff, _redact, _match_label) do NOT need try/except —
if they throw, it's a bug, let it crash.

### 9.3 Consumer contract

Consumers MUST use try/except/finally:

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

---

## 10. Decision — Logging Contract

### 10.1 Manager logging

Every ensure() outcome logs these fields:

| Log site | match_keys | uuid | before | after | changed | duration_ms |
|---|---|---|---|---|---|---|
| create (+ check_mode) | yes | yes* | — | yes | yes | yes |
| update (+ check_mode) | yes | yes | yes | yes | yes | yes |
| delete (+ check_mode) | yes | yes | yes | — | yes | yes |
| noop | yes | yes | — | — | yes | yes |
| error | yes | yes* | yes* | — | — | yes |
| ambiguous_match | yes | all | — | — | — | — |
| validation_failed | — | — | — | — | — | — |

Severity: DEBUG=noop, INFO=create/update, WARNING=delete, ERROR=failure/ambiguity/validation.

`duration_ms` uses `time.monotonic()` — zero overhead, no sleeps.

### 10.2 Client logging

Every HTTP call:

| Outcome | Level | Fields |
|---|---|---|
| Success | DEBUG | method, endpoint, status_code, duration_ms |
| Retry | WARNING | method, endpoint, status_code, attempt, duration_ms |
| Failure | ERROR | method, endpoint, status_code, duration_ms |

### 10.3 Structured logging

Use **structlog** with:
- Colorized console output for development
- JSON file output for Loki/Promtail ingestion
- Redaction processor applied before both outputs
- `duration_ms` on every log entry

---

## 11. Decision — Testing

### 11.1 Test structure — per concern per manager

```
tests/
├── unit/
│   ├── core/
│   │   ├── test_identity.py        # IdentityResolver unit tests
│   │   ├── test_diff.py            # DiffEngine unit tests
│   │   ├── test_redaction.py       # Redactor unit tests
│   │   ├── test_validation.py      # Validator unit tests
│   │   ├── test_endpoint.py        # EndpointResolver unit tests
│   │   └── test_logging_helpers.py # ManagerLogBuilder unit tests
│   ├── test_auth_user.py           # AuthUserManager CRUD
│   ├── test_auth_user_validation.py # AuthUser field validation
│   ├── test_fw_filter.py           # FwFilter CRUD
│   ├── test_fw_filter_ambiguous.py # FwFilter duplicate detection
│   └── ...
├── integration/
│   ├── test_auth_lifecycle.py      # Auth domain lifecycle
│   ├── test_fw_lifecycle.py        # Firewall domain lifecycle
│   ├── test_fw_ambiguous.py        # Duplicate detection on live device
│   └── ...
└── smoke/
    └── test_smoke.py               # Import + instantiation only
```

One test file per concern per manager. Small files, clear scope.

### 11.2 Error handling tests (mandatory per ADR-0029 §10)

Every manager test suite MUST include:

1. create/update/delete failure: logs ERROR, re-raises unchanged
2. Exception type preserved through manager layer
3. FieldValidationError raised before API call for bad params
4. AmbiguousMatchError raised with correct UUIDs on duplicate
5. Consumer try/except/finally pattern works correctly
6. Transport errors (401, 403, 404, 400, 500, timeout, connection)

### 11.3 Quality gates

| Gate | Tool | Command |
|---|---|---|
| Lint | ruff | `ruff check src/ tests/` |
| Format | ruff | `ruff format --check src/ tests/` |
| Types | mypy | `mypy src/ --ignore-missing-imports` |
| Security | bandit | `bandit -r src/ -q -ll` |
| Unit tests | pytest | `pytest tests/unit/ -q` |
| CVE scan | pip-audit | `pip-audit` |

**ALL gates must pass locally before commit.** CI confirms — but the developer/agent
verifies first.

Python version matrix: 3.10, 3.11, 3.12, 3.13.

---

## 12. Decision — Git Flow for Libraries

### 12.1 Branch + PR for everything

- **Never push directly to main** — always branch + PR
- **Never merge without CI green** — check `gh run list` after push
- **Use PR template** — every PR uses `.github/PULL_REQUEST_TEMPLATE.md` with ALL sections filled
- **Use commit template** — conventional commits with file list, test count, endpoint list

### 12.2 Merge strategy (per ADR-0019)

- `gh pr merge N --merge --delete-branch` (--no-ff merge commit)
- Never `--squash` (destroys commit history — disabled in GitHub settings)
- Never `--rebase` (loses PR boundary — disabled in GitHub settings)
- Contributor rebases locally before push

### 12.3 No partial work

- No PR without all quality gates passing
- No manager without validators
- No validators without unit tests
- No integration test without safety boundaries documented
- No merge without CI confirmed green

---

## 13. Decision — Packaging

- **Build system:** `pyproject.toml` with `hatchling`
- **Install method:** `pip install git+https://github.com/{org}/{repo}.git@{tag}`
- **Future:** GitLab CE PyPI registry when deployed
- **Versioning:** Conventional Commits + release-please
- **Runtime deps:** httpx, structlog (minimal)
- **Dev deps:** ruff, mypy, pytest, pytest-cov, pytest-asyncio, bandit, pip-audit
- **Devcontainer:** `.devcontainer/devcontainer.json` with Python + ruff + mypy

---

## 14. Decision — Secret Redaction

Each manager declares `REDACT_FIELDS` set. Redaction is unified in `core/redaction.py`:

```python
REDACT_RULES = {
    "password":       (0, 0, "*"),   # full — <REDACTED:password>
    "secret":         (0, 4, "*"),   # last 4 — ***...ExQi
    "api_key":        (4, 4, "*"),   # first 4 + last 4 — gTCJ***igCn
    "authorizedkeys": (8, 0, "*"),   # first 8 — ssh-ed25***
}
```

Rules:
- Passwords, OTP seeds: always full redaction — never show any part
- API keys, tokens: partial reveal for verification
- Redaction runs as structlog processor + in EnsureResult before/after fields
- `core/redaction.py` is the single source — logging.py delegates to it

---

## 15. Consequences

- All `lib-*` repositories MUST follow this standard
- lib-opnsense requires a refactor to extract core/ components (scope of work exists)
- lib-synology-dsm requires alignment audit
- New libraries start compliant from day one
- Ansible modules consume the lib via `pip install git+...@tag`, never git clone

---

## 16. References

- ADR-0029 v1 (archived): `archive/0029-python-library-design-standard.v1.md`
- ADR-0019: Git Workflow and Approval Process
- lib-opnsense refactor scope: `lib-opnsense/docs/refactor-scope-of-work.md`
- lib-opnsense match key strategy: `lib-opnsense/docs/api-match-key-reference.md`
- OPNsense MVC Field Types: https://docs.opnsense.org/development/frontend/models_fieldtypes.html
- OPNsense MVC Constraints: https://docs.opnsense.org/development/frontend/models_constraints.html
- OPNsense Firewall API: https://docs.opnsense.org/development/api/core/firewall.html
