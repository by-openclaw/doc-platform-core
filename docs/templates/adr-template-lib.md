<!--
  SCOPE GUARD — LIB DEV ADR
  ==========================
  This template is for decisions internal to a software library.
  DO NOT use this template for:
  - Infrastructure decisions (VLANs, VMs, TLS PKI)
  - Platform-wide policies (those belong in doc-platform-core)
  - Credential storage format for infra services
  Wrong template = PR blocked.
-->

# ADR-XXXX: Title in sentence case

<!-- One line. Sentence case. No trailing period. Example: "Ensure pattern for resource manager methods" -->

**Status:** Draft | Accepted | Deprecated | Superseded by ADR-XXXX
**Date:** YYYY-MM-DD
**Deciders:** @yboujraf

---

## Context

<!--
  What problem is this solving within this library?
  - Which lib component is affected: client / manager / model / all?
  - What was wrong or missing before this decision?
  - Reference prior lib ADRs if relevant.
  Keep to 2–5 sentences or a short list. No infrastructure context here.
-->

## Decision

<!--
  One clear statement of what was decided.
  Start with a bold summary sentence, then detail below.
  Example: "**All manager methods expose an `ensure()` that returns `EnsureResult(changed, action)`.**"
-->

## Dependency Injection

<!--
  How are dependencies passed into this component?
  - Constructor injection / factory / config object — pick one, explain why
  - No hidden globals, no module-level singletons, no monkey-patchable state
  - Every dependency must be injectable for testing without live infrastructure
  Show the expected constructor/factory signature if helpful.
-->

## External Dependencies

<!--
  List any new external packages introduced by this decision.
  Bias: fewer deps > more deps. If stdlib covers it, use stdlib.
  For each new dep:
  - Package name + version pin
  - SPDX license
  - Justification: why this package, not an alternative
  - Alternatives considered and rejected (brief)
  If no new deps, write "No new external dependencies introduced."
-->

| Package | Version | SPDX license | Justification | Alternatives rejected |
|---------|---------|--------------|---------------|-----------------------|
|         |         |              |               |                       |

## API Contract

<!--
  What external API endpoints does this decision touch or introduce?
  - REST method, path, version
  - Swagger/OpenAPI reference path if one exists in the lib docs
  - Input schema: required fields, types, constraints
  - Output schema: structure, types, error envelope
  If this ADR does not change the API surface, write "No API contract changes."
-->

| Method | Path | Input | Output | OpenAPI ref |
|--------|------|-------|--------|-------------|
|        |      |       |        |             |

## Idempotency

<!--
  How does this decision satisfy the ensure/idempotent pattern?
  - `ensure()` MUST return `EnsureResult(changed: bool, action: str)`
  - Running twice against the same target must produce identical state and the same result structure
  - Describe what "no change needed" looks like vs "change applied"
  - Describe what "change failed" propagates (exception type, not silent return)
  This section is MANDATORY — there is no opt-out from the ensure pattern.
-->

## Separation of Concerns

<!--
  Which layer owns this decision? Be explicit.
  Layer model:
    client   → auth, HTTP transport, session management, raw API calls
    manager  → business logic, ensure/idempotent orchestration, error mapping
    model    → data schema, serialization, validation
  Rules:
  - No business logic in client
  - No HTTP calls in manager (use injected client)
  - No mutable state in model
  State which layer this ADR modifies and why that layer is the correct owner.
-->

## OOB Handling

<!--
  What happens when the target device/service is unreachable?
  - Define the error path explicitly: exception type, message, caller contract
  - Silent failures are banned — the caller must know when an operation could not complete
  - Timeouts: what is the connect timeout? read timeout? [OWNER TO DEFINE if not already set]
  - Partial failure: if a batch operation fails mid-way, what is the rollback or state signal?
-->

## Error Mapping

<!--
  How are errors surfaced to callers?
  - List ERROR_MAP entries if this lib uses a central error map
  - List exception types raised (custom or stdlib)
  - Error codes: format, uniqueness guarantee
  - Do NOT re-raise raw HTTP or vendor exceptions to callers — wrap them
-->

| Error condition | Exception type | Error code | Caller action |
|-----------------|---------------|------------|---------------|
|                 |               |            |               |

## Log Format

<!--
  Define structured log fields for all log events introduced by this decision.
  Rules:
  - No credentials in logs — ever
  - No PII in logs — ever
  - Correlation ID passed through if available, not generated per-call
  - Log level: DEBUG (trace/internal), INFO (state change), WARNING (recoverable error), ERROR (unrecoverable)
  - Log sink: stdout (default) — no file logging unless explicitly required
-->

| Event | Level | Fields | Notes |
|-------|-------|--------|-------|
|       |       |        |       |

## Cross-Cutting Concerns

<!--
  Address each of the following. If not applicable, state "N/A — [reason]".
  Do NOT skip silently.
-->

**Auth:**
<!-- Auth is handled in the client layer only. This section confirms that — or explains any exception. -->

**Retry:**
<!-- Strategy (linear / exponential backoff), max attempts, which error codes trigger retry, which do not. [OWNER TO DEFINE backoff values if not already set in lib config] -->

**Rate-limiting:**
<!-- Is the target API rate-limited? If yes: requests/minute/hour, how does this lib respect that limit? -->

**Correlation IDs:**
<!-- Are correlation IDs passed through call chains? If yes: field name, propagation mechanism. If no: explain why not needed. -->

## Unit Tests

<!--
  What must unit tests cover for this decision?
  - Coverage floor: [OWNER TO DEFINE — typically 100% for manager methods]
  - Mocking strategy: no live calls in unit tests — mock at the client boundary
  - List the critical test cases (happy path, idempotent no-op, error paths, OOB)
  CI must enforce the coverage floor. If it doesn't yet, note it as a gap.
-->

**Coverage floor:** [OWNER TO DEFINE]

**Required test cases:**
- Happy path: ...
- Idempotent no-op (second call, no change): ...
- Error path (target returns error): ...
- OOB (target unreachable): ...

**Mocking strategy:**
<!-- How is the client mocked? Fixture, mock library, test double? -->

## Integration Tests

<!--
  Integration tests require a live target. Define clearly:
  - What target is required (NAS IP, credentials, device model)
  - Test matrix: what operations are covered
  - Skip conditions: tests MUST be marked skip (not fail) when target is unavailable
    Use: pytest.mark.skipif / equivalent — do NOT use xfail for infrastructure unavailability
  - Do NOT run integration tests in CI unless a live target is provisioned in the CI environment
-->

**Target required:** <!-- e.g. Synology DS1513+ at 10.1.x.x, DSM 7.x -->

**Test matrix:**

| Test | Operation | Expected result |
|------|-----------|-----------------|
|      |           |                 |

**Skip condition:** `<!-- pytest.mark.skipif(not TARGET_AVAILABLE, reason="...") -->`

## Dev Container

<!--
  Every lib must ship a dev container that works on Win11, macOS, and Linux via Docker Desktop.
  No WSL required on Windows — Docker Desktop handles the Linux layer.
  Confirm this decision's changes are reflected in the devcontainer config.
-->

| Component         | Value / Status |
|-------------------|----------------|
| Base image        |                |
| Language runtime  |                |
| Linter            |                |
| Formatter         |                |
| Test runner       |                |
| Debug config      |                |
| postCreateCommand | <!-- Must install pre-commit hooks --> |
| Win11 tested      | Yes / No / N/A |
| macOS tested      | Yes / No / N/A |
| Linux tested      | Yes / No / N/A |

## CISO

<!--
  Data classification and auth model for this lib decision.
  Rules:
  - No plaintext credentials in code, logs, or test fixtures — ever
  - Auth model must be explicit: where credentials enter the call chain (client layer only)
  - Data classification: what sensitivity level do inputs/outputs carry?
    Classifications: public | internal | confidential | secret
-->

| Item                | Detail |
|---------------------|--------|
| Input data class    | <!-- public / internal / confidential / secret --> |
| Output data class   | <!-- public / internal / confidential / secret --> |
| Auth model          | <!-- API key / session token / OAuth / other — handled in: client layer --> |
| Credential in logs? | No — confirmed |
| PII in scope?       | <!-- Yes (describe) / No --> |

## Consequences

<!--
  What does this decision enable within the lib? What does it constrain?
  List known risks. If there's a known gap or future work required, say so.
  Structure: positive consequences first, then constraints, then risks.
-->

**Enables:**
-

**Constrains:**
-

**Known risks:**
-

## References

<!--
  External links: API docs, RFCs, library docs, security advisories.
  Internal: related lib ADRs, doc-platform-core ADRs if cross-cutting.
-->

-
