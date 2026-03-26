# Scope of Work — `{{PROJECT_NAME}}`

- **Document ID:** `sow-{{PROJECT_SLUG}}-{{SUBJECT}}-{{DATE}}`
- **Customer:** `{{CUSTOMER_NAME}}`
- **Customer slug:** `{{CUSTOMER_SLUG}}`
- **Prepared by:** `BY-SYSTEMS`
- **Date:** `{{DATE}}`
- **Version:** `{{VERSION}}`
- **Status:** `Draft`

---

## 1. Executive Summary

{{PROJECT_NAME}} is a BY-SYSTEMS delivery scoped to implement, configure, integrate, or document the following outcome:

> {{EXECUTIVE_SUMMARY}}

This work aligns with the BY-SYSTEMS reference platform:

- open source, self-hosted, low lock-in
- standard DevOps operating model
- GitLab-centered CI/CD
- Vault-backed secret management
- Traefik as edge proxy
- NetBox as infrastructure source of truth
- observability through Prometheus, Grafana, Loki, and related tooling

---

## 2. Objectives

The objectives of this engagement are:

- {{OBJECTIVE_1}}
- {{OBJECTIVE_2}}
- {{OBJECTIVE_3}}

Success means the agreed deliverables are completed, tested, documented, and handed over according to the acceptance criteria in this document.

---

## 3. Scope

### In Scope

- {{IN_SCOPE_1}}
- {{IN_SCOPE_2}}
- {{IN_SCOPE_3}}

### Out of Scope

- {{OUT_SCOPE_1}}
- {{OUT_SCOPE_2}}
- {{OUT_SCOPE_3}}

Any item not explicitly listed under **In Scope** is considered out of scope unless approved through change control.

---

## 4. Delivery Context

### Platform Tier

- `Tier 1` — Base platform
- `Tier 2` — Optional module _(broadcast / voip / cctv / other)_

### Target Environments

Use one of the approved BY-SYSTEMS environment tracks:

```text
dev → test → staging → acceptance → prod
```

or for lighter projects:

```text
dev → staging → prod
```

### Naming and Standards

All repo, branch, document, FQDN, image, and service names must follow the BY-SYSTEMS naming convention:

- lowercase only
- hyphens only
- English only
- no spaces or underscores

Examples:

- repo: `platform-vault-config`
- branch: `feat/42-add-vault-integration`
- document: `sow-infra-network-setup-2026-03-25.md`
- FQDN: `vault.by-systems.internal`

---

## 5. Technical Scope

### Components Included

| Domain | Component | Notes |
|---|---|---|
| Compute | `{{COMPUTE_COMPONENT}}` | {{COMPUTE_NOTES}} |
| IaC | `{{IAC_COMPONENT}}` | {{IAC_NOTES}} |
| Networking | `{{NETWORK_COMPONENT}}` | {{NETWORK_NOTES}} |
| Identity | `{{IDENTITY_COMPONENT}}` | {{IDENTITY_NOTES}} |
| Monitoring | `{{MONITORING_COMPONENT}}` | {{MONITORING_NOTES}} |
| Security | `{{SECURITY_COMPONENT}}` | {{SECURITY_NOTES}} |
| Docs | `{{DOC_COMPONENT}}` | {{DOC_NOTES}} |

Remove rows that do not apply.

### Standard BY-SYSTEMS Stack References

Depending on scope, the implementation may include or integrate with:

- Proxmox VE
- k3s
- Terraform
- Ansible
- pfSense
- WireGuard / NetBird
- Traefik
- step-ca
- GitLab CE + Runner + Registry
- Vault
- Authentik
- Vaultwarden
- PostgreSQL / Redis / MinIO
- NetBox / Neo4J
- Prometheus / Grafana / Loki / Zabbix
- Wazuh / OpenSCAP / Falco / Trivy / Gitleaks

Optional modules:

- broadcast: PTP, AES67, SMPTE ST 2110, VRF RED/BLUE
- voip: Kamailio, RTPEngine, Coturn, Homer
- cctv: Frigate or ZoneMinder

---

## 6. Deliverables

The engagement will produce the following deliverables:

1. `{{DELIVERABLE_1}}`
2. `{{DELIVERABLE_2}}`
3. `{{DELIVERABLE_3}}`
4. `{{DELIVERABLE_4}}`

Typical deliverables may include:

- infrastructure code repositories
- Ansible playbooks / Terraform modules
- CI/CD pipeline configuration
- service deployment manifests or Helm values
- runbooks and ADRs
- diagrams (`.drawio`, `.puml`, exported `.png`)
- handover documentation and acceptance report

---

## 7. Assumptions

This scope assumes:

- customer stakeholders are available for validation and access approvals
- required network, DNS, firewall, and infrastructure prerequisites exist or are separately planned
- BY-SYSTEMS standard tooling is accepted unless an exception is documented
- credentials, licenses, and API access required from the customer are provided on time
- deviations from the reference architecture are documented and approved

Add project-specific assumptions here:

- {{ASSUMPTION_1}}
- {{ASSUMPTION_2}}

---

## 8. Dependencies

Dependencies may include:

- customer infrastructure readiness
- DNS delegation / Cloudflare access if public naming is required
- VPN connectivity (WireGuard / NetBird)
- access to hypervisor, Kubernetes, or firewall platforms
- availability of identity source / SSO integration
- hardware delivery for Arista / UniFi / pfSense / servers if relevant

Project-specific dependencies:

- {{DEPENDENCY_1}}
- {{DEPENDENCY_2}}

---

## 9. Acceptance Criteria

The work is considered accepted when:

- agreed deliverables are completed
- deployment or configuration is validated in the target environment
- tests defined for the engagement pass
- documentation is delivered and reviewed
- known limitations are recorded
- customer sign-off is received

Optional detailed criteria:

| Area | Acceptance Criterion |
|---|---|
| Functionality | {{AC_1}} |
| Security | {{AC_2}} |
| Operations | {{AC_3}} |
| Documentation | {{AC_4}} |

---

## 10. Exclusions and Constraints

### Constraints

- {{CONSTRAINT_1}}
- {{CONSTRAINT_2}}

### Explicit Exclusions

- 24/7 managed service operations unless separately contracted
- custom features not described in this SOW
- commercial third-party tooling outside approved exceptions
- undocumented architecture drift from the BY-SYSTEMS baseline

---

## 11. Risks

| Risk | Impact | Mitigation |
|---|---|---|
| {{RISK_1}} | {{RISK_1_IMPACT}} | {{RISK_1_MITIGATION}} |
| {{RISK_2}} | {{RISK_2_IMPACT}} | {{RISK_2_MITIGATION}} |

---

## 12. Roles and Responsibilities

| Role | Responsibility | Owner |
|---|---|---|
| Project lead | Coordination, planning, reporting | {{OWNER_1}} |
| Technical lead | Architecture and technical decisions | {{OWNER_2}} |
| Customer representative | Validation and sign-off | {{OWNER_3}} |

---

## 13. Change Control

Any change to scope, assumptions, schedule, deliverables, or acceptance criteria must:

1. be documented in writing
2. include impact on cost, timeline, and risk
3. be reviewed by BY-SYSTEMS and the customer
4. be approved before implementation

---

## 14. Documentation and Handover

The delivery should include, where applicable:

- README and operator notes
- runbooks for backup, restore, upgrade, and incident response
- ADRs for major design decisions
- diagrams in editable and exported form
- environment-specific deployment notes
- credentials handover process via approved secret-sharing workflow

Recommended documentation paths:

```text
docs/adr/
docs/diagrams/
docs/runbooks/
```

---

## 15. Sign-Off

| Name | Role | Signature | Date |
|---|---|---|---|
| {{SIGNOFF_1}} | BY-SYSTEMS |  |  |
| {{SIGNOFF_2}} | Customer |  |  |

---

## Appendix A — Reference Naming Patterns

```text
Repo:      {scope}-{component}-{qualifier}
Branch:    {type}/{issue-id}-{short-description}
Commit:    {type}({scope}): {description}
Document:  {type}-{subject}-{YYYY-MM-DD}.md
FQDN:      {service}.by-systems.internal
Device:    {function}-{vendor}-{env}-{number:02d}
```

## Appendix B — Reference Internal Services

Typical internal FQDNs:

- `gitlab.by-systems.internal`
- `vault.by-systems.internal`
- `authentik.by-systems.internal`
- `netbox.by-systems.internal`
- `grafana.by-systems.internal`

Use environment-qualified names when needed:

- `gitlab.dev.by-systems.internal`
- `gitlab.staging.by-systems.internal`
- `gitlab.prod.by-systems.internal`
