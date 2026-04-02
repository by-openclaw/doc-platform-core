# Generic PR Template — All Repos

> **Status:** DRAFT — share to Rune for implementation across all 5 repos
> **Principle:** Generic now, refine per repo as we progress (agile, not waterfall)

---

## PR Template (replace in all repos)

```markdown
## Summary

<!-- What changed and why. One sentence. -->

Closes #___

## Type

- [ ] Feature
- [ ] Bug fix
- [ ] Documentation
- [ ] Chore / refactor
- [ ] Security

## Verification

<!-- Check what applies to THIS change. Not all boxes apply to every PR. -->

### Code quality (if code changed)
- [ ] Linting clean (ruff / ansible-lint / terraform fmt)
- [ ] Type checking clean (mypy — if applicable)
- [ ] Unit tests pass
- [ ] Integration tests pass (if touching live infra/NAS)
- [ ] Coverage maintained (no drop)

### Security (always)
- [ ] No secrets, tokens, or passwords in committed files
- [ ] No `<REDACTED>` values in code (only in docs)
- [ ] ADR compliance section present (if new ADR)

### Documentation (if applicable)
- [ ] CHANGELOG entry added
- [ ] CLAUDE.md updated (if state changed)
- [ ] RAID.md updated (if new risk/issue found)

### Idempotency (if automation/lib code)
- [ ] ensure() returns EnsureResult with correct action
- [ ] dry_run=True tested
- [ ] Running twice produces same result

## Review

- [ ] @by-opus review requested (label: `review:opus`)
```

---

## Issue Template — Task (add to all repos)

```yaml
# .github/ISSUE_TEMPLATE/task.yml
name: Task
description: Sprint task — approved work item
labels: ["task"]
body:
  - type: input
    id: needs-ref
    attributes:
      label: NEEDS.md reference
      description: Which NEEDS.md item does this implement?
      placeholder: "H11, M5, C1, etc."
    validations:
      required: true
  - type: dropdown
    id: assigned-agent
    attributes:
      label: Assigned to
      options:
        - "@by-rune"
        - "@by-opus"
        - "@yboujraf"
    validations:
      required: true
  - type: textarea
    id: scope
    attributes:
      label: Scope of work
      description: What exactly needs to be done? Clear, no ambiguity.
    validations:
      required: true
  - type: textarea
    id: done-criteria
    attributes:
      label: Definition of Done
      description: How do we know this is complete?
    validations:
      required: true
```

---

## Release template (already handled by Release Please — no change needed)

Release Please auto-generates from conventional commits. No manual template.

---

*Share to Rune. He applies the PR template to all 5 repos and adds task.yml issue template. Then we focus on infra.*
