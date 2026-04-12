# ADR Refactor — DRAFT

> **Status:** DRAFT — review per scope before any content is moved or merged.
> **Date:** 2026-04-12
> **Archive:** All 34 originals saved as `archive/NNNN-*.pre-refactor.md`

---

## Proposed Structure

```
docs/adr/
├── identity/
│   └── 0001-identity-access-model.md        ← merge 0004+0024+0031+0033
├── git/
│   └── 0001-workflow-and-release.md          ← merge 0005+0019+0034
├── naming/
│   └── 0001-naming-convention.md             ← merge 0010+0028+0030
├── security/
│   ├── 0001-secret-storage.md                ← merge 0011+0016
│   ├── 0002-compliance-mapping.md            ← keep 0013
│   ├── 0003-hardening.md                     ← keep 0021
│   └── 0004-certificate-strategy.md          ← keep 0014
├── infra/
│   ├── 0001-platform-stack.md                ← keep 0001
│   ├── 0002-platform-charter.md              ← keep 0006
│   ├── 0003-terraform-standard.md            ← keep 0008
│   ├── 0004-network-architecture.md          ← merge 0015+0032
│   ├── 0005-environment-tiers.md             ← keep 0012
│   ├── 0006-logging.md                       ← keep 0017
│   ├── 0007-monitoring.md                    ← keep 0023
│   └── 0008-backup-strategy.md               ← keep 0020
├── lib/
│   └── python/
│       └── 0001-design-standard.md           ← keep 0029
├── services/
│   ├── 0001-opnsense-provisioning.md         ← keep 0027
│   ├── 0002-email-infrastructure.md          ← keep 0026
│   ├── 0003-netbox-cmdb.md                   ← keep 0009
│   ├── 0004-database-strategy.md             ← keep 0018
│   └── 0005-licensing-policy.md              ← keep 0022
└── archive/                                  ← all flat originals
```

---

## Scope-by-Scope Review Checklist

Each scope must be reviewed and approved before execution.

### 1. identity/ — PENDING REVIEW

| Source ADR | Title | Action |
|---|---|---|
| 0004 | Identity & SSO architecture | **Target** — base document |
| 0024 | Identity provisioning sync standard | Merge into 0004 |
| 0031 | CI token identity standard | Merge into 0004 |
| 0033 | User identity access standard | Merge into 0004 |

**Result:** `identity/0001-identity-access-model.md` — one comprehensive identity ADR.

---

### 2. git/ — PENDING REVIEW

| Source ADR | Title | Action |
|---|---|---|
| 0005 | VCS and CI/CD strategy | **Split** — migration/strategy = ADR, CI details = OS §5 |
| 0019 | Git workflow approval process | **Target** — base document |
| 0034 | Git configuration standard | Merge into 0019 |

**Result:** `git/0001-workflow-and-release.md` — unified git workflow ADR.
**Side effect:** CI procedure parts of 0005 move to OPERATING-STANDARD.md §5.

---

### 3. naming/ — PENDING REVIEW

| Source ADR | Title | Action |
|---|---|---|
| 0010 | Naming & identity convention | **Target** — base document |
| 0028 | Firewall alias/rule naming convention | Merge into 0010 |
| 0030 | Automation naming convention | Merge into 0010 |

**Result:** `naming/0001-naming-convention.md` — sub-sections: infra, software, automation, network.
**Note:** 0002 (repo structure) — naming part merges here, structure part → OS §3.

---

### 4. security/ — PENDING REVIEW

| Source ADR | Title | Action |
|---|---|---|
| 0011 | Secret storage convention | **Target** — base for secret-storage |
| 0016 | Vault KV path convention | Merge into 0011 |
| 0013 | Compliance framework mapping | Keep as-is → `security/0002` |
| 0021 | Hardening standard | Keep as-is → `security/0003` |
| 0014 | Certificate strategy | Keep as-is → `security/0004` |

**Result:** 4 security ADRs (1 merged, 3 kept).

---

### 5. infra/ — PENDING REVIEW

| Source ADR | Title | Action |
|---|---|---|
| 0001 | Platform stack decisions | Keep as-is → `infra/0001` |
| 0006 | Platform charter | Keep as-is → `infra/0002` |
| 0008 | Terraform state management | Keep as-is → `infra/0003` |
| 0015 | Network VLAN architecture | **Target** — base for network |
| 0032 | Network zone VLAN registry | Merge into 0015 |
| 0012 | Environment tier standard | Keep as-is → `infra/0005` |
| 0017 | Logging standard | Keep as-is → `infra/0006` |
| 0023 | Monitoring approach | Keep as-is → `infra/0007` |
| 0020 | Backup strategy | Keep as-is → `infra/0008` |

**Result:** 8 infra ADRs (1 merged, 7 kept).

---

### 6. lib/ — PENDING REVIEW

| Source ADR | Title | Action |
|---|---|---|
| 0029 | Python library design standard | Keep as-is → `lib/python/0001` |

**Result:** 1 ADR. Future: `lib/ansible/0001` when needed.

---

### 7. services/ — PENDING REVIEW

| Source ADR | Title | Action |
|---|---|---|
| 0027 | OPNsense provisioning contract | Keep as-is → `services/0001` |
| 0026 | Email infrastructure standard | Keep as-is → `services/0002` |
| 0009 | NetBox CMDB | Keep as-is → `services/0003` |
| 0018 | Database strategy | Keep as-is → `services/0004` |
| 0022 | Licensing policy | Keep as-is → `services/0005` |

**Result:** 5 service ADRs (all kept as-is, just relocated).

---

## Archive (not architecture decisions)

| Source ADR | Title | Reason |
|---|---|---|
| 0003 | Issue tracking standard | Already in OS §7 |
| 0007 | Automation scripting standard | Already in OS (every script → a repo) |
| 0025 | Notification/collaboration | Config, not architecture |

---

## Split to OS (procedure, not architecture)

| Source ADR | Part | Destination |
|---|---|---|
| 0002 | Repo structure rules | OS §3 |
| 0005 | CI pipeline details | OS §5 |
| 0019 | PR template | OS (already there) |

---

## Rules

1. One repo (`doc-platform-core`) holds ALL platform ADRs
2. Scoped by concern folder, numbered per-scope (identity/0001, git/0001, etc.)
3. Before creating a new ADR → check if it fits an existing scope
4. Repos reference central ADRs, never duplicate
5. Repo `CLAUDE.md` has implementation choices (httpx, urllib), not ADRs
6. Old flat ADRs archived in `archive/`, never deleted
7. All repos referencing old ADR numbers must be updated after each scope

---

## Execution Order (proposed)

1. **identity/** — highest impact (4 ADRs → 1), agents most confused here
2. **git/** — second most fragmented (3 ADRs → 1)
3. **naming/** — third most fragmented (3 ADRs → 1)
4. **security/** — 1 merge + 3 relocations
5. **infra/** — 1 merge + 7 relocations
6. **services/** — 5 relocations only
7. **lib/** — 1 relocation only
8. **Cleanup** — update README, update all repo references, archive flat files
