> **ARCHIVED 2026-03-30** — Most gaps resolved as of v0.8.0/v0.9.3/v0.10.0. See `lib-synology-dsm/docs/` for current state and `lib-synology-dsm/docs/audits/` for the 2026-03-30 full audit.

# lib-synology-dsm — Gap Report & Fix Plan

**Date:** 2026-03-29
**Version audited:** v0.6.1
**Repo:** `by-openclaw/lib-synology-dsm`
**Auditor:** Rune (DevOps familiar, BY-SYSTEMS)
**Scope:** Production readiness + Ansible collection porting readiness

---

## Executive Summary

The library has functional CRUD for User, Group, Share, NFS, and FileStation. It is **not production-ready**. The core logic works; the scaffolding required for a library to be *trustworthy* in automation pipelines is largely absent: no unit test coverage for the actual managers, no exception hierarchy, no `dry_run` support, mypy fails on 10 errors, the integration tests don't run through `pytest` properly, and the Vault provider has a dead parameter and no live test coverage.

Total gaps identified: **19**

---

## 1. `ensure()` / Idempotent Pattern

**Assessment:** Partially present — the pattern exists but has correctness gaps.

| Gap | Severity | Effort |
|---|---|---|
| `UserManager.ensure(state="present")` always calls `update()` even when nothing changed — `changed: True` on every run with an existing user | **High** | 30 min |
| `ShareManager.ensure(state="present")` same — unconditional update → always reports changed | **High** | 30 min |
| `GroupManager.ensure(state="present")` same — unconditional update → always reports changed | **High** | 30 min |
| `NFSManager` has no `ensure()` at all — only `get_rules()` (stub) | **Critical** | 2 h |
| `FileStationManager` has no `ensure()` — no idempotent folder/file existence check | **High** | 1 h |
| `ensure()` return value does not include a `before`/`after` diff — Ansible modules need it | **Medium** | 1 h |
| `ShareManager.ensure()` does not accept `owner_user`, `owner_group`, or NFS params — `create_with_permissions()` exists but is not wired to `ensure()` | **High** | 1.5 h |

**Detail on ensure() correctness:**

```python
# Current (wrong):
if name not in existing:
    self.create(...)
    return {"changed": True, "action": "created"}
else:
    self.update(name, description=description, email=email)  # always runs
    return {"changed": True, "action": "updated"}  # always True

# Correct (needs):
# Fetch current state, compare, only update if diff, return changed=False if noop
```

---

## 2. `dry_run` Support

**Assessment:** Completely absent across the entire library.

| Gap | Severity | Effort |
|---|---|---|
| `ensure()` on all managers — no `dry_run=True` parameter | **Critical** | 2 h |
| `FileStationManager.upload()` — no dry run; will overwrite files on live NAS silently | **Critical** | 1 h |
| `FileStationManager.delete()` — no dry run; irreversible operation, no safeguard | **Critical** | 30 min |
| `ShareManager.delete()` — no dry run | **High** | 30 min |
| `UserManager.delete()` / `GroupManager.delete()` — no dry run | **Medium** | 30 min |
| No `check_mode` adapter for Ansible module compatibility | **High** | 1 h |

**Note:** Without `dry_run`, the library cannot be safely integrated into Ansible in `--check` mode. Every Ansible module wrapping this library would need to implement its own guards, duplicating logic.

---

## 3. Unit Tests

**Assessment:** Critically insufficient — only 2 unit tests exist, both for `DSMClient.login()` only.

**Run result:**
```
tests/test_client.py::test_login_success PASSED
tests/test_client.py::test_login_failure PASSED
2 passed in 0.13s
```

**Integration tests run through pytest erroneously** — `tests/integration/test_live_nas.py` is structured as a standalone script with a `__main__` block, but pytest collects its `test_*` functions and they fail due to missing fixtures:

```
ERROR tests/integration/test_live_nas.py::test_users — fixture 'sid' not found
ERROR tests/integration/test_live_nas.py::test_groups — fixture 'sid' not found
ERROR tests/integration/test_live_nas.py::test_shares — fixture 'sid' not found
ERROR tests/integration/test_live_nas.py::test_filestation — fixture 'sid' not found
```

CI (`pytest tests/ -v`) currently exits non-zero. The CI workflow is broken.

| Gap | Severity | Effort |
|---|---|---|
| Zero unit tests for `UserManager`, `GroupManager`, `ShareManager`, `NFSManager`, `FileStationManager` | **Critical** | 4–6 h |
| Zero unit tests for `ensure()` state machine (present/absent/already-exists/already-gone) | **Critical** | 3 h |
| Integration tests (`test_live_nas.py`) are discovered by pytest but fail fixture resolution — CI is broken | **Critical** | 1 h |
| `tests/test_client.py` missing `tests/unit/` directory structure — no `conftest.py`, no shared mocks | **Medium** | 30 min |
| `test_auth_admin` returns a value instead of asserting — pytest warns `PytestReturnNotNoneWarning` | **Low** | 10 min |
| No `tests/unit/` directory at all | **High** | 30 min (scaffold) |

**Missing test scenarios:**

- `UserManager.create()` → mock response → assert payload shape
- `UserManager.ensure(state="present")` when user exists → assert no create called
- `UserManager.ensure(state="absent")` when user missing → assert changed=False
- `GroupManager.add_member()` → assert fetch-then-merge logic
- `ShareManager.create()` → assert `shareinfo` JSON format
- `FileStationManager.upload()` → assert SynoToken in URL query, cookie set, field name `path`
- `VaultCredentialProvider.get()` with mocked hvac → assert field mapping

---

## 4. Integration Tests

**Assessment:** The bash smoke tests are solid. The Python integration test has structural issues.

### Bash smoke tests

| File | Coverage |
|---|---|
| `tests/integration/dsm-crud-test.sh` | Auth, User CRUD, Group CRUD + membership, Share CRUD + NFS + permissions, cleanup |
| `tests/integration/test_full_crud.sh` | Same scope, older structure, uses `grep` not `jq` |
| `tests/integration/dsm-share-reference.sh` | Share reference only |

| Gap | Severity | Effort |
|---|---|---|
| **FileStation ops completely absent from all bash smoke tests** — no upload/download/list/delete via curl | **Critical** | 2 h |
| `test_full_crud.sh` uses `grep -q '"success":true'` (fragile, no jq) — could false-positive | **Medium** | 30 min |
| No smoke test for `VaultCredentialProvider` — Vault integration is untested at the integration level | **High** | 1 h |
| Hardcoded credentials in bash scripts (`BySyst3ms_`, `BySyst3ms_test!`) — secrets in test files | **High** | 30 min |

**Missing FileStation curl commands (for reference when implementing):**

```bash
# List shares
curl -sk "https://${NAS}:${PORT}/webapi/entry.cgi" \
  -H "X-SYNO-TOKEN: ${TOKEN}" \
  --data "_sid=${SID}&api=SYNO.FileStation.List&version=2&method=list_share"

# List files in folder
curl -sk "https://${NAS}:${PORT}/webapi/entry.cgi" \
  -H "X-SYNO-TOKEN: ${TOKEN}" \
  --data "_sid=${SID}&api=SYNO.FileStation.List&version=2&method=list&folder_path=/by-terraform-state"

# Create folder (mkdir)
curl -sk "https://${NAS}:${PORT}/webapi/entry.cgi" \
  -H "X-SYNO-TOKEN: ${TOKEN}" \
  --data "_sid=${SID}&api=SYNO.FileStation.CreateFolder&version=2&method=create" \
  --data-urlencode 'folder_path=["/by-terraform-state"]' \
  --data-urlencode 'name=["rune-test-dir"]' \
  --data "force_parent=true"

# Upload file (multipart — SynoToken in URL, session as cookie id=)
curl -sk "https://${NAS}:${PORT}/webapi/entry.cgi?api=SYNO.FileStation.Upload&method=upload&version=2&SynoToken=${TOKEN}" \
  -H "X-Syno-Token: ${TOKEN}" \
  --cookie "id=${SID}" \
  -F "path=/by-terraform-state/rune-test-dir" \
  -F "create_parents=true" \
  -F "overwrite=true" \
  -F "file=@/tmp/test-upload.txt;type=application/octet-stream"

# Download file
curl -sk "https://${NAS}:${PORT}/webapi/entry.cgi" \
  -H "X-SYNO-TOKEN: ${TOKEN}" \
  --data "_sid=${SID}&api=SYNO.FileStation.Download&version=2&method=download&mode=download" \
  --data-urlencode "path=/by-terraform-state/rune-test-dir/test-upload.txt" \
  -o /tmp/test-download.txt

# Delete file
curl -sk "https://${NAS}:${PORT}/webapi/entry.cgi" \
  -H "X-SYNO-TOKEN: ${TOKEN}" \
  --data "_sid=${SID}&api=SYNO.FileStation.Delete&version=2&method=start&accurate_progress=false" \
  --data-urlencode "path=/by-terraform-state/rune-test-dir/test-upload.txt"
```

---

## 5. Code Quality

### ruff

```
src/synology_dsm/filestation.py:149: F401 `urllib.parse` imported but unused
1 fixable error
```

Minor. One line fix.

### mypy (10 errors across 5 files)

| Error | File | Severity | Notes |
|---|---|---|---|
| `Cannot find implementation or library stub for module named "dotenv"` | credentials.py | Low | Optional dep — `--ignore-missing-imports` masks it in CI |
| `Library stubs not installed for "hvac"` | credentials.py | Low | Optional dep — same |
| `Function "synology_dsm.shares.ShareManager.list" is not valid as a type` | shares.py:326 | **High** | Method named `list` shadows Python builtin `list` type in type annotations — `existing = {s["name"] for s in self.list()}` causes mypy confusion |
| `Function "synology_dsm.groups.GroupManager.list" is not valid as a type` | groups.py:145 | **High** | Same — `list` method name collision with builtin |
| `Function "synology_dsm.users.UserManager.list" is not valid as a type` | users.py:35, 147 | **High** | Same |
| `"list?[builtins.dict[Any, Any]]" has no attribute "__iter__"` | groups.py:110, 134 | **High** | Downstream of the `list` naming issue |
| `Dict entry 0 has incompatible type "str": "str \| None"` | filestation.py:120 | Medium | `self._c._synotoken` can be `None` or empty string — needs null guard |

**Root cause for majority of mypy errors:** Naming a method `list()` shadows the Python builtin type `list` within the class scope. All three managers (`UserManager`, `GroupManager`, `ShareManager`) have this issue. The fix is to rename to `list_all()` or add type: ignore, or restructure annotations.

### Docstrings (PEP 257)

- `UserManager`: ✅ all public methods documented
- `GroupManager`: ✅ all public methods documented
- `ShareManager`: ✅ all public methods documented
- `FileStationManager`: ✅ all public methods documented
- `NFSManager`: ⚠️ only `get_rules()` — a stub with no docstring on the class
- `DSMClient`: ✅ documented
- `credentials.py`: ✅ documented

| Gap | Severity | Effort |
|---|---|---|
| `urllib.parse` unused import in filestation.py | **Low** | 1 min |
| 10 mypy errors — CI uses `--ignore-missing-imports` masking real type errors | **High** | 2 h |
| `list()` method name collides with builtin, breaks mypy type inference across 3 managers | **High** | 1 h |
| `filestation.py` — `_synotoken` could be None/empty, dict type incompatibility | **Medium** | 30 min |
| `NFSManager` class missing class-level docstring | **Low** | 5 min |

---

## 6. Error Handling

**Assessment:** Flat `RuntimeError` with string messages everywhere. No exception hierarchy.

| Gap | Severity | Effort |
|---|---|---|
| All API errors raise `RuntimeError(f"API error [{api}.{method}]: {data.get('error')}")` — callers cannot distinguish auth errors from permission errors from resource-not-found | **Critical** | 3 h |
| No exception hierarchy — missing `DSMError`, `DSMAuthError` (error 400), `DSMPermissionError` (error 403), `DSMNotFoundError`, `DSMAPIError` | **Critical** | 2 h |
| DSM error codes (400=wrong credentials, 402=disabled, 403=permission denied, 119=session expired) are embedded in string messages — not inspectable programmatically | **Critical** | included above |
| `VaultCredentialProvider.get()` in `get_credentials()` catches bare `except Exception: pass` — Vault auth failures silently fall through to env-based auth with no warning | **High** | 30 min |
| `NFSManager.get_rules()` — stub that will fail at runtime (uses `SYNO.Core.Share.NFS` which doesn't exist on DSM 7.x / DS1513+, correct API is `SYNO.Core.FileServ.NFS.SharePrivilege`) | **Critical** | 30 min |
| `GroupManager.list_members()` catches bare `RuntimeError` as fallback — masks real errors | **Medium** | 30 min |

**Example of what's needed:**

```python
class DSMError(Exception):
    def __init__(self, message: str, code: int | None = None):
        super().__init__(message)
        self.code = code

class DSMAuthError(DSMError): pass       # code 400, 402
class DSMPermissionError(DSMError): pass  # code 403
class DSMNotFoundError(DSMError): pass
class DSMSessionError(DSMError): pass     # code 119

# In client.py:
error = data.get("error", {})
code = error.get("code") if isinstance(error, dict) else None
if code in (400, 402):
    raise DSMAuthError(f"Auth failed: {error}", code=code)
elif code == 403:
    raise DSMPermissionError(f"Permission denied: {error}", code=code)
```

---

## 7. Vault Integration

**Assessment:** Structurally present, functionally unvalidated, has dead code.

| Gap | Severity | Effort |
|---|---|---|
| `vault_role` parameter accepted in `VaultCredentialProvider.__init__()` but never used in `get()` — dead code | **Medium** | 15 min |
| No AppRole auth support — only token auth (not suitable for production automation; tokens expire/rotate) | **High** | 2 h |
| No Kubernetes auth support (relevant for k3s deployment) | **Medium** | 2 h |
| `get_credentials()` silently falls back to env if Vault fails — failure mode is invisible in logs | **High** | 30 min |
| Zero integration tests against a live Vault instance — Vault path format (`secret/data/...`) is assumed but untested | **High** | 2 h |
| No mocked unit test for `VaultCredentialProvider.get()` | **Medium** | 1 h |
| Missing: Vault policy example in docs — callers don't know what Vault policy to write for the secret path | **Medium** | 30 min |

---

## 8. Ansible Portability

**Assessment:** The `ensure(state=present/absent)` pattern is in place on 3 of 5 managers, but not Ansible-ready.

| Gap | Severity | Effort |
|---|---|---|
| No `dry_run` / `check_mode` pass-through — Ansible modules cannot run in `--check` mode | **Critical** | 2 h |
| `ensure()` always returns `changed=True` on updates even if nothing changed — Ansible idempotency is broken | **Critical** | 2 h |
| No `diff` output from `ensure()` — Ansible `--diff` mode needs before/after state | **High** | 2 h |
| `NFSManager` and `FileStationManager` have no `ensure()` — cannot be wrapped as Ansible resources | **High** | 3 h |
| `ShareManager.ensure()` does not accept `owner_user`, `owner_group`, `nfs_client` — the full share lifecycle requires `create_with_permissions()` which is separate and not idempotent | **High** | 2 h |
| No `ansible_collections/` skeleton or module examples in repo | **Low** | 4 h |
| Exception hierarchy missing — Ansible modules need to catch `DSMPermissionError` vs `DSMAuthError` separately to set `module.fail_json(msg=...)` correctly | **Critical** | included in §6 |

---

## 9. Bash Smoke Test — FileStation Coverage

**Assessment:** FileStation is entirely absent from all bash smoke tests.

`dsm-crud-test.sh` covers: Auth, User CRUD, Group CRUD + membership, Share CRUD + NFS + permissions.

**FileStation is not covered at all.** No upload, no download, no list, no mkdir, no delete via curl.

The curl-equivalent commands for all FileStation operations are documented in §4 above.

| Gap | Severity | Effort |
|---|---|---|
| FileStation section missing from `dsm-crud-test.sh` | **High** | 1.5 h |
| FileStation upload curl pattern is non-standard (SynoToken in URL, cookie auth, multipart) — easy to get wrong; reference test would prevent regression | **High** | included above |

---

## Summary Table

| Area | Critical | High | Medium | Low |
|---|---|---|---|---|
| ensure() / idempotency | 1 | 5 | 1 | 0 |
| dry_run | 3 | 2 | 0 | 1 |
| Unit tests | 3 | 1 | 1 | 2 |
| Integration tests | 1 | 2 | 1 | 0 |
| Code quality | 0 | 3 | 2 | 2 |
| Error handling | 4 | 1 | 2 | 0 |
| Vault integration | 0 | 3 | 3 | 0 |
| Ansible portability | 3 | 3 | 0 | 1 |
| Bash smoke test | 0 | 2 | 0 | 0 |
| **Total** | **15** | **22** | **10** | **6** |

---

## Top 3 Blockers — Production Readiness & Ansible Collection Porting

### Blocker 1 — No Exception Hierarchy (Critical)

Every error surfaces as `RuntimeError` with a string. Callers (and Ansible modules) cannot distinguish "wrong credentials" from "permission denied" from "share already exists" without parsing error messages. Until `DSMError`/`DSMAuthError`/`DSMPermissionError` exist, any automation built on this library is fragile.

**Fix estimate:** 3–4 h (new `exceptions.py`, update `client.py`, update `NFSManager` stub)

### Blocker 2 — `ensure()` Reports `changed=True` on No-Op Updates + No `dry_run` (Critical)

The idempotency contract is broken: `ensure(state="present")` always calls `update()` and returns `changed=True` even if the resource already matches the desired state. Combined with the complete absence of `dry_run=True`, this makes:

1. CI pipelines report false changes on every run
2. Ansible `--check` mode impossible to implement
3. Ansible idempotency tests always fail

**Fix estimate:** 4–5 h (fetch-compare-diff pattern in all `ensure()` methods + `dry_run` parameter)

### Blocker 3 — CI is Broken + Zero Manager Unit Tests (Critical)

`pytest tests/` exits non-zero in CI due to `test_live_nas.py` being collected as a pytest suite but having no pytest fixtures. Only 2 unit tests exist (login happy/sad path) — zero coverage for any manager method. A library cannot be published to a package registry, integrated into Ansible, or trusted in production without a working test suite.

**Fix estimate:** 6–8 h (move integration tests behind `pytest.mark.integration` + add `tests/unit/` with mocked manager tests)

---

## Recommended Fix Sequence

Priority order for reaching `v1.0.0-rc1` (production + Ansible ready):

1. **Fix CI immediately** — add `pytest.ini` or `pyproject.toml` marker to skip integration tests by default (30 min)
2. **Exception hierarchy** — `exceptions.py` + update `client.py` (3 h)
3. **Fix `list()` naming** — rename to `list_all()` or add proper type hints; clears 8 of 10 mypy errors (1 h)
4. **Fix `ensure()` idempotency** — fetch-compare-noop pattern + `changed=False` when state matches (3 h)
5. **Add `dry_run` to `ensure()`** and destructive methods (2 h)
6. **Unit tests for all managers** — mocked, no NAS required (5 h)
7. **FileStation ensure + smoke test** — `ensure()` for folders + bash FileStation section (3 h)
8. **NFSManager stub fix** — wire to correct API + add `ensure()` (2 h)
9. **Vault: AppRole auth + silent-failure fix** (2 h)
10. **`ShareManager.ensure()` wire to `create_with_permissions()`** (2 h)

**Total estimated remediation effort:** ~24 h

---

*Report generated by Rune (DevOps familiar, BY-SYSTEMS) — 2026-03-29*
*Do NOT implement fixes based on this report without review — audit only.*
