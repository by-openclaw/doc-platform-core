# GitHub Identity Setup — Agent Accounts & Token Management

> **Date:** 2026-04-01
> **Status:** DONE — accounts created, tokens active, documented for reproducibility
> **Scope:** How to create, configure, and maintain GitHub agent identities for BY-SYSTEMS

---

## 1. Current State

| GitHub user | Role | Token scopes | Repo permission | Status |
|---|---|---|---|---|
| `@yboujraf` | Owner | Full (classic PAT) | Admin | ✅ Active |
| `@by-rune` | Executor | `repo`, `workflow`, `project`, `read:org` | Maintain | ✅ Active |
| `@by-opus` | Auditor | `repo`, `project`, `read:org` | Triage | ✅ Active |

Tokens stored in: `workspace/infra/secrets/ci-github-{agent}.json`

---

## 2. What Rune needs to do NOW

### 2a. Switch to @by-rune identity for git operations

```bash
# On Rune VM — configure git to use @by-rune
git config --global user.name "by-rune"
git config --global user.email "github-bot+rune@by-systems.be"

# Set token for git push (HTTPS)
# Option 1: environment variable
export GH_TOKEN=$(jq -r '.fields.token' ~/.openclaw/workspace/infra/secrets/ci-github-rune.json)

# Option 2: git credential helper
git config --global credential.helper store
# Then on first push, use: username=by-rune, password=<token from ci-github-rune.json>
```

### 2b. Update repo secrets for CI workflows

Release Please uses `secrets.GH_TOKEN` in each repo. Currently set to @yboujraf's PAT (which may be expired — causing CI failures).

**Decision needed:** Which token should CI workflows use?

| Workflow | Should use | Why |
|---|---|---|
| `release-please.yml` | `@by-rune` PAT | Rune creates releases — commits should show as @by-rune |
| `project-board-sync.yml` | `@yboujraf` PAT (`PROJECT_TOKEN`) | Org project access requires org admin |

**Rune executes:**

```bash
# Read token from secret file
RUNE_TOKEN=$(jq -r '.fields.token' ~/.openclaw/workspace/infra/secrets/ci-github-rune.json)

# Update GH_TOKEN in all 5 repos (for Release Please)
for repo in lib-synology-dsm doc-platform-core ansible-platform infra-terraform-proxmox platform-setup; do
  gh secret set GH_TOKEN --repo by-openclaw/$repo --body "$RUNE_TOKEN"
done

# PROJECT_TOKEN stays as @yboujraf's PAT (already set at org level)
```

### 2c. Verify CI is green after token switch

```bash
# Trigger a test push on one repo, check workflow runs
gh run list --repo by-openclaw/lib-synology-dsm --limit 3
```

---

## 3. Opus token usage

Opus uses `@by-opus` token for **read-only audit operations:**

```bash
# Opus reads PRs, issues, CI logs — never pushes
export GH_TOKEN=$(jq -r '.fields.token' ~/.openclaw/workspace/infra/secrets/ci-github-opus.json)

gh pr list --repo by-openclaw/lib-synology-dsm
gh pr view 42 --repo by-openclaw/lib-synology-dsm
gh run view 12345 --repo by-openclaw/lib-synology-dsm --log-failed
gh issue create --repo by-openclaw/lib-synology-dsm --title "Audit: ..." --label "audit"
```

Opus can create issues (Triage permission) but cannot push code or merge PRs.

---

## 4. Token storage — centralized

All tokens in one place: `workspace/infra/secrets/`

| File | Agent | Scopes |
|---|---|---|
| `ci-github-pat.json` | `@yboujraf` | Full (org admin, project board) |
| `ci-github-rune.json` | `@by-rune` | `repo`, `workflow`, `project`, `read:org` |
| `ci-github-opus.json` | `@by-opus` | `repo`, `project`, `read:org` |

**Rule:** Agents read their own token file. Never read another agent's token. Never share tokens in chat.

---

## 5. Reproducible process — How to create a new agent account

### Step 1: Create GitHub account

```
1. Go to https://github.com/signup
2. Email: github-bot+{agent-name}@by-systems.be
3. Username: by-{agent-name}
4. Password: generate in Vaultwarden, min 16 chars
5. Verify email
6. Do NOT enable 2FA (agents can't handle OTP interactively)
```

### Step 2: Invite to org

```
1. Log in as @yboujraf
2. Go to https://github.com/orgs/by-openclaw/people
3. Click "Invite member"
4. Username: by-{agent-name}
5. Role: Member
6. Send invitation
7. Log in as by-{agent-name} → accept invitation
```

### Step 3: Set repo permissions

```
For each repo in the org:
1. Go to github.com/by-openclaw/{repo}/settings/access
2. Add by-{agent-name}
3. Role:
   - Executor agents → Maintain
   - Auditor agents → Triage
   - Read-only agents → Read
```

### Step 4: Generate PAT

```
1. Log in as by-{agent-name}
2. Settings → Developer settings → Personal access tokens → Tokens (classic)
3. Note: "by-{agent-name}-agent-{YYYY-MM} — {role} — by-openclaw org"
4. Expiration: 90 days (or custom — document in secret file)
5. Scopes:
   - Executor: repo, workflow, project, read:org
   - Auditor: repo, project, read:org
   - Read-only: repo (read), read:org
6. Generate → copy token
```

### Step 5: Store token

```
Create: workspace/infra/secrets/ci-github-{agent-name}.json

{
  "vault_path": "secret/ci/github/{agent-name}",
  "description": "GitHub PAT for @by-{agent-name} — {role}",
  "access": "{rw|ro}",
  "owner": "by-{agent-name}",
  "target": "github.com — by-openclaw org",
  "github_account": {
    "username": "by-{agent-name}",
    "email": "github-bot+{agent-name}@by-systems.be",
    "role": "{role description}",
    "org_role": "Member",
    "repo_permission": "{Maintain|Triage|Read}"
  },
  "fields": {
    "token": "{paste-token-here}",
    "scopes": "{scopes}",
    "note": "Classic PAT — agent: by-{agent-name}, org: by-openclaw"
  }
}
```

### Step 6: Update secrets README

Add the new agent to `workspace/infra/secrets/README.md` — Access table + Files table.

### Step 7: Configure agent

```
On agent VM:
  git config user.name "by-{agent-name}"
  git config user.email "github-bot+{agent-name}@by-systems.be"
  export GH_TOKEN=$(jq -r '.fields.token' ~/.openclaw/workspace/infra/secrets/ci-github-{agent-name}.json)
```

### Step 8: Update CI repo secrets (if executor)

```bash
TOKEN=$(jq -r '.fields.token' ~/.openclaw/workspace/infra/secrets/ci-github-{agent-name}.json)
for repo in lib-synology-dsm doc-platform-core ansible-platform infra-terraform-proxmox platform-setup; do
  gh secret set GH_TOKEN --repo by-openclaw/$repo --body "$TOKEN"
done
```

---

## 6. Token rotation policy

| Token | Rotation | Reminder |
|---|---|---|
| Agent PATs | Every 90 days | GitHub sends expiry email to agent's email |
| @yboujraf PAT | Every 90 days | Same |
| Repo secrets (GH_TOKEN) | When agent PAT rotates | Must update after every rotation |

**When rotating:**
1. Generate new PAT (same scopes)
2. Update `ci-github-{agent}.json` with new token
3. Update repo secrets: `gh secret set GH_TOKEN --repo ... --body "$NEW_TOKEN"`
4. Verify CI green
5. Revoke old PAT in GitHub Settings

---

## 7. Compliance

| Framework | Control | How this satisfies it |
|---|---|---|
| ISO 27001 | A.9.2.1 (user registration) | Documented process for account creation |
| ISO 27001 | A.9.2.3 (privileged access) | Maintain vs Triage vs Read — least privilege |
| ISO 27001 | A.9.2.5 (access review) | Token rotation every 90 days |
| ISO 27001 | A.9.4.3 (password management) | PATs stored in Vault-ready JSON, min 16 chars |
| NIS2 | Art.21(2)(i) (HR security) | Separate identities per agent, audit trail |

---

*This document is the runbook. Follow it for any new agent account. ADR to formalize when top 3 ADRs are written.*
