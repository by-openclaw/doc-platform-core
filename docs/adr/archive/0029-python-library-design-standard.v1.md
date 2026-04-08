# ADR-0029 — Python Library Design Standard

**Date:** 2026-04-04
**Status:** Accepted
**Deciders:** Youssef Boujraf
**Tags:** `python`, `lib`, `design-standard`, `ensure`, `asyncio`

---

## 1. Status

Accepted

---

## 2. Context

BY-SYSTEMS builds Python libraries (`lib-*` repos) to automate infrastructure products — Synology DSM, OPNsense, and future targets. Each library wraps a vendor API and exposes idempotent resource management to Ansible, Terraform, and CI pipelines. Without a shared design standard, each library invents its own patterns for client transport, error handling, async execution, and idempotency. This causes inconsistent caller contracts, duplicated design effort, and friction when onboarding new products.

This ADR defines the mandatory design standard for all `lib-*` repositories.

---

## 3. Decision — OOP Architecture

All `lib-*` libraries follow a manager-based OOP architecture:

- **Manager pattern:** A `BaseManager` abstract base class defines the interface. One concrete manager per domain (e.g. `AuthUserManager`, `FwAliasManager`). Managers own all business logic and the `ensure()` contract.
- **Client class:** `{Product}Client` — handles transport, authentication, and session lifecycle. Managers receive a client instance via constructor injection.
- **Models:** Frozen dataclasses per entity (e.g. `AuthUser`, `FwAlias`). Models carry data only — no business logic, no API calls.
- **Exceptions:** Typed hierarchy rooted in a product-scoped base exception. Minimum set: base, auth, permission, not-found, validation, timeout, connection. Each product maps vendor error codes to the appropriate typed exception.

---

## 4. Decision — Dependency Injection

All configuration is supplied via constructor parameters. Nothing is hardcoded.

**Client constructor parameters:**

| Parameter | Purpose |
|---|---|
| `host` | Target hostname or IP |
| `port` | Target port |
| `verify_ssl` | TLS certificate verification |
| `timeout` | Per-request HTTP timeout (seconds) |
| `async_timeout` | Timeout for `asyncio.gather()` operations (seconds) |
| `max_retries` | Maximum retry attempts on transient failure |
| `retry_backoff` | Backoff factor between retries (seconds) |

**Credential provider pattern:** Credentials are never stored in client constructors. A credential provider abstraction supplies them at runtime:

- `EnvCredentialProvider` — reads from environment variables
- `VaultCredentialProvider` — reads from HashiCorp Vault (AppRole or token auth)

Providers implement a common interface returning a credentials object. The client accepts a provider instance, not raw username/password strings.

---

## 5. Decision — Separation of Concerns

Every `lib-*` repository follows this source layout:

```
src/{product}/
├── client.py          <- transport only (HTTP, auth, session)
├── exceptions.py      <- typed errors only
├── models/            <- dataclasses only (no logic)
│   ├── auth_user.py
│   └── ...
├── managers/          <- business logic + ensure()
│   ├── base.py        <- BaseManager ABC
│   ├── auth_user.py
│   └── ...
└── credentials.py     <- credential providers (DI)
```

**Rules:**

- `client.py` handles HTTP transport, session management, authentication headers, and error-code-to-exception mapping. It contains zero business logic.
- `exceptions.py` defines the typed exception hierarchy. Nothing else lives here.
- `models/` contains frozen dataclasses. No imports from `client`, `managers`, or `credentials`.
- `managers/` contains all business logic, the `ensure()` contract, diff computation, and secret redaction. Managers depend on `client` and `models` only.
- `credentials.py` defines credential providers. No imports from `managers` or `models`.

---

## 6. Decision — Async / Await (NOT polling, NOT threading)

All `lib-*` libraries use Python `asyncio` with `async/await` for concurrent operations:

- **Client provides both sync and async interfaces.** Sync callers use the sync interface; Ansible and CI pipelines use async where supported.
- **`httpx.AsyncClient`** for async HTTP. httpx supports both sync (`httpx.Client`) and async (`httpx.AsyncClient`) in a single dependency.
- **`asyncio.gather()`** for parallel operations — equivalent to `Promise.all()` in TypeScript.
- **`asyncio.wait_for(coro, timeout=N)`** enforces hard deadlines. No infinite loops are possible.
- **`async with` context managers** ensure resource cleanup. No resource leaks.
- **Long-running operations** (e.g. firmware install): `await client.wait_for_ready(timeout=300)` uses `asyncio.sleep` between polls, not blocking `time.sleep`.

**Example — parallel ensure():**

```python
async def provision_auth(client):
    user_mgr = AuthUserManager(client)
    group_mgr = AuthGroupManager(client)

    # Parallel execution -- like Promise.all()
    user_result, group_result = await asyncio.gather(
        user_mgr.ensure("svc-monitoring", state="present"),
        group_mgr.ensure("grp-monitoring", state="present"),
    )
```

**Note:** Libraries that have a hard zero-dependency constraint (e.g. lib-synology-dsm uses stdlib `urllib` only per its ADR-0001) are exempt from the httpx requirement but must still expose async interfaces via `asyncio`.

---

## 7. Decision — ensure() Contract

The `ensure()` pattern from lib-synology-dsm ADR-0002 is promoted to a platform-wide standard.

Every manager MUST implement:

```python
async def ensure(
    self,
    name: str,
    state: Literal["present", "absent"] = "present",
    check_mode: bool = False,
    **desired_fields,
) -> EnsureResult:
```

**Behaviour contract:**

1. **Fetch current state** — list existing resources, build a lookup map.
2. **Diff** — compare desired fields against current state.
3. **Act only if diff** — if state already matches, return `EnsureResult(changed=False, action="noop")`.
4. **check_mode=True** — return what would happen, make zero API calls. This is the dry-run mode.
5. **Return `EnsureResult`** — a dataclass with: `changed` (bool), `action` (str), `before` (dict | None), `after` (dict | None), `diff` (dict | None).

**Action strings:** `"created"`, `"updated"`, `"deleted"`, `"noop"`, `"would_create"`, `"would_update"`, `"would_delete"`.

**Idempotency guarantee:** Running `ensure()` twice with the same parameters produces `changed=False` on the second run. No re-application, no duplicate resources.

---

## 8. Decision — Error Handling

- **Typed exception hierarchy.** Never raise generic `Exception`. Every library defines a product-scoped base exception (e.g. `DSMError`, `OPNError`) with typed subclasses for auth, permission, not-found, validation, timeout, and connection errors.
- **Retry with configurable policy.** `max_retries` and `retry_backoff` are injected via the client constructor. Retry logic lives in the client transport layer, not in managers.
- **Timeout via `asyncio.wait_for()`.** Hard deadline on any coroutine. Raises `asyncio.TimeoutError` — never hangs.
- **No infinite loops** anywhere in the codebase. All loops have a bounded iteration count or a timeout guard.
- **No bare `except:`.** Always catch specific exception types.
- **Resource cleanup via `async with` / context managers.** Clients implement `__aenter__` / `__aexit__` (async) and `__enter__` / `__exit__` (sync) for guaranteed session teardown.

### try / except / finally — mandatory pattern

Every mutation operation (create, update, delete, assign) in managers MUST use:

```python
# Inside manager — log + re-raise
try:
    uuid = await self._client.create(endpoint, payload_key, params)
    await self._apply()
except Exception as exc:
    logger.error("create failed: %s", exc, extra={"action": "create_failed", ...})
    raise
```

Every consumer (Ansible module, script, integration test) MUST use:

```python
# Consumer — catch typed exceptions, never crash
try:
    result = await mgr.ensure("present", params)
except ProductValidationError as exc:
    logger.error("Validation: %s", exc.validations)
    # handle gracefully — report to user, skip, retry
except ProductAuthError:
    logger.critical("Auth failed — check API key")
    # abort — do not retry auth failures
except ProductTimeoutError:
    logger.warning("Timeout — will retry")
    # retry or escalate
except ProductError as exc:
    logger.error("API error: %s", exc)
    # generic fallback
finally:
    # ALWAYS runs — cleanup resources, close connections, report status
    # Use for: closing files, releasing locks, sending notifications
    pass
```

Rules:
- **`try`**: wrap the operation that may fail
- **`except`**: catch most-specific exception first, then broader (validation before base)
- **`finally`**: guaranteed cleanup — runs on success AND failure. Use for resource teardown, audit logging, status reporting
- **Never swallow exceptions silently** — always log at ERROR or re-raise
- **Never use bare `except:`** — always specify the exception type
- **Managers re-raise after logging** — they don't decide recovery strategy, the consumer does

---

## 9. Decision — Secret Redaction

- Each manager declares a `REDACT_FIELDS` set listing field names that contain secrets (e.g. `{'password', 'privkey', 'psk', 'secret', 'otp_seed'}`).
- Redaction applies in all logs, diffs, and `EnsureResult.diff` output.
- Raw API responses containing secret fields are never logged. The client transport layer does not log response bodies; managers redact before any logging or result construction.

### Partial reveal with configurable depth

Full redaction (`<REDACTED:field>`) prevents verification. Use partial reveal where safe:

```python
REDACT_RULES = {
    "password":       (0, 0, "*"),   # full — <REDACTED:password>
    "secret":         (0, 4, "*"),   # last 4 — ***...ExQi
    "api_key":        (4, 4, "*"),   # first 4 + last 4 — gTCJ***igCn
    "authorizedkeys": (8, 0, "*"),   # first 8 — ssh-ed25***
}
# Rule: (reveal_start, reveal_end, mask_char)
# Values shorter than reveal window → full redaction (safe default)
```

Rules:
- **Passwords, OTP seeds**: always full redaction `(0, 0)` — never show any part
- **API keys, tokens**: partial reveal `(4, 4)` — first + last 4 chars for verification
- **SSH keys**: show type prefix `(8, 0)` — enough to identify key type
- **Secrets**: show last 4 `(0, 4)` — enough to verify which secret is in use
- Redaction runs as a **structlog processor** — applied before both console and file output

---

## 10. Decision — Testing

| Category | Requirements |
|---|---|
| Unit tests | Mocked client (`pytest` + `unittest.mock`), no network access |
| Integration tests | Live device required, marked with `@pytest.mark.integration` |
| Smoke tests | Import + instantiation only — verifies packaging |
| Ansible playbooks | One `playbook_{scope}.yml` per domain for integration validation |
| Error handling tests | **Mandatory** — every test suite must verify try/except/finally behavior |

**Error handling test requirements (mandatory for all managers):**

Every manager test suite (unit AND integration) must include error handling tests that verify:

1. **Manager logs ERROR and re-raises** — create/update/delete failures emit structured log, then re-raise the original exception unchanged.
2. **Exception type is preserved** — `OpnsenseValidationError` stays `OpnsenseValidationError` through the manager, not wrapped in a generic `OpnsenseError`.
3. **Consumer try/except/finally** — test the full consumer pattern: except catches typed exceptions, finally always runs (client.close()).
4. **Invalid state raises ValueError** — `ensure(state="running", ...)` must raise `ValueError` immediately.
5. **Auth error** — bad credentials produce `OpnsenseAuthError` (401), not a generic exception.
6. **Connection/timeout error** — unreachable host produces `OpnsenseConnectionError` or `OpnsenseTimeoutError`.
7. **Validation error** — invalid params produce `OpnsenseValidationError` with `.validations` dict.

> **Rationale:** Error handling failures are silent — they only surface in production when a real error occurs.
> Without explicit tests, a broken re-raise or swallowed exception goes undetected until an incident.

**Quality gates:**

- PEP 8 enforced via `ruff`
- Type hints enforced via `mypy` (`disallow_untyped_defs = true`)
- Docstrings mandatory on all public methods
- Coverage fail-under: 80%

---

## 11. Decision — CI Pipeline

Every `lib-*` repository runs the following CI stages:

| Stage | Tool | Command |
|---|---|---|
| Lint | ruff | `ruff check` + `ruff format --check` |
| Types | mypy | `mypy src/` |
| Tests | pytest | `pytest tests/unit/ --cov --cov-fail-under=80` |
| Security (SAST) | bandit | `bandit -r src/` |
| Security (CVE) | pip-audit | `pip-audit` |

**Python version matrix:** 3.10, 3.11, 3.12, 3.13.

**Integration gate:** Optional CI stage. Requires live device credentials stored as CI secrets. Not required for merge; required for release.

---

## 12. Decision — Packaging and Versioning

- **Build system:** `pyproject.toml` with `hatchling` build backend.
- **Runtime dependencies:** Minimal. `httpx` for async HTTP where permitted. Zero-dep libs (stdlib urllib only) are valid where explicitly decided per-repo.
- **Dev dependencies:** `ruff`, `mypy`, `pytest`, `pytest-cov`, `bandit`, `pip-audit`.
- **Versioning:** Conventional Commits + release-please. No manual version bumps. No commitizen on repos using release-please.
- **Devcontainer:** VS Code + Python 3.13 + ruff + mypy + Swagger viewer extensions. Defined in `.devcontainer/devcontainer.json`.

---

## 13. Consequences

- All `lib-*` repositories MUST follow this standard from the date of acceptance.
- Existing `lib-synology-dsm` (v1) aligns with most decisions already. The v2 WIP branch is designed to comply fully.
- New libraries (e.g. `lib-opnsense`) start compliant from day one — no retrofit needed.
- Non-compliant managers in existing libs are tracked as tech debt and resolved before the next major release.

---

## 14. References

- ADR-0007: Automation and Scripting Standard — separation of concerns, idempotency requirements
- ADR-0019: Git Workflow and Approval Process — Conventional Commits, release-please, branch naming
- lib-synology-dsm ADR-0002: Idempotent ensure() pattern — promoted to platform standard by this ADR
- lib-synology-dsm `src/synology_dsm/exceptions.py` — reference exception hierarchy implementation
- lib-synology-dsm `src/synology_dsm/client.py` — reference client pattern implementation
- Python asyncio documentation: https://docs.python.org/3/library/asyncio.html
