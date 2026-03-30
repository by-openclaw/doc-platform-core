# Security Policy

## Supported Versions

| Version | Supported |
|---|---|
| 0.5.x | ✅ Current |
| < 0.5 | ❌ No fixes |

## Reporting a Vulnerability

**Do not open a public GitHub issue for security vulnerabilities.**

Report privately via: security@by-systems.be

Include:
- Description of the issue (e.g., exposed secrets in documentation)
- Location (file path and line number)
- Impact assessment

We aim to acknowledge reports within 48 hours and provide a fix within 14 days for confirmed issues.

## Scope

This is a documentation-only repo. Security considerations:
- Documentation may reference internal infrastructure (IPs, hostnames, architecture)
- Secrets must never appear in documentation — use `<REDACTED:{type}>` format
- ADRs and RAID entries may reference security-sensitive decisions
- All repos are private — documentation is not publicly accessible
