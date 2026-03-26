# BY-SYSTEMS Platform — Idempotency and Strategy Guide
**Last updated:** 2026-03-26
**Status:** Active
**Scope:** All platform tools
**Related docs:** `docs/stack.md`, `docs/netbox.md`, `docs/roadmap.md`

> **Principle:** Every automation action must be safe to run repeatedly. Running a playbook, pipeline, or script twice must produce the same result as running it once.
> **Source of truth:** NetBox for infrastructure state. Git for code and configuration. Vault for secrets.

---

## 1. Infrastructure as Code

### Terraform (Proxmox Provider)

| Aspect | Strategy |
|---|---|
| **State** | Remote state in MinIO S3 backend (`s3://terraform-state/{scope}/`) |
| **Idempotency** | Built-in — `terraform apply` converges to declared state |
| **Drift detection** | `terraform plan` shows delta between state and reality |
| **Free plan** | Terraform CLI is MPL 2.0 (free). No Terraform Cloud needed. |
| **NetBox integration** | Post-apply: Ansible registers created VMs in NetBox |

**Pattern:**
```hcl
# Proxmox VM — declarative, idempotent
resource "proxmox_vm_qemu" "gitlab" {
  name        = "vm-gitlab-prod-01"
  target_node = "srv-proxmox-prod-01"
  clone       = "debian-12-base"  # Packer golden image
  cores       = 4
  memory      = 8192
}
```

**Anti-patterns to avoid:**
- Never use `local-exec` provisioners for config management — use Ansible
- Never store state locally — always use remote backend
- Never hardcode IPs — allocate in NetBox IPAM, reference via data source or variable

### Ansible

| Aspect | Strategy |
|---|---|
| **Inventory** | Dynamic from NetBox (`netbox.netbox.nb_inventory`) |
| **Idempotency** | Built-in for most modules. Avoid `command`/`shell` unless idempotent (`creates`/`removes` guards). |
| **Free plan** | Ansible core is GPL v3 (free). No AAP/Tower needed. Use Semaphore for web UI if needed. |
| **Secrets** | `hashi_vault` lookup plugin — never in playbooks or inventory |
| **NetBox integration** | Dynamic inventory + `netbox.netbox` modules for registration |

**Idempotent patterns:**

```yaml
# Good: module-based, declarative
- name: Ensure Nginx is installed
  ansible.builtin.apt:
    name: nginx
    state: present

# Good: file with template (idempotent via checksum)
- name: Deploy config
  ansible.builtin.template:
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf
  notify: restart nginx

# Bad: shell command without guard
- name: Install package
  ansible.builtin.shell: apt-get install -y nginx
  # NOT idempotent — runs every time

# Acceptable: shell with creates guard
- name: Init database
  ansible.builtin.shell: /opt/app/init-db.sh
  args:
    creates: /opt/app/.db-initialized
```

**Ansible + NetBox flow:**
1. Ansible reads NetBox dynamic inventory
2. Playbook runs against discovered hosts
3. Post-task: Ansible updates NetBox (version, status, tags)
4. Result: NetBox always reflects current state

### Arista EOS (Network Automation)

| Aspect | Strategy |
|---|---|
| **Collection** | `arista.eos` via Ansible |
| **Idempotency** | EOS modules are declarative — `state: present` / `state: replaced` |
| **Dry run** | `ansible-playbook --check --diff` previews changes |
| **Free plan** | Arista EOS API (eAPI) is built-in, no license needed |

```yaml
- name: Configure VLAN
  arista.eos.eos_vlans:
    config:
      - vlan_id: 10
        name: mgmt
        state: active
    state: merged  # idempotent merge
```

---

## 2. CI/CD

### GitLab CE (Self-Hosted)

| Aspect | Strategy |
|---|---|
| **Idempotency** | Pipelines are triggered by commits; same commit = same pipeline |
| **Free plan** | GitLab CE (MIT license). Unlimited repos, runners, registries. |
| **Limitations vs EE** | No advanced RBAC, no compliance pipelines, no DORA metrics |
| **NetBox integration** | CI updates NetBox `version` custom field on deploy |

**Pipeline idempotency pattern:**
```yaml
deploy:
  script:
    - helm upgrade --install $SERVICE ./charts/$SERVICE
      --namespace $NAMESPACE
      --values values/$ENV.yaml
      --atomic  # rolls back on failure
      --wait
```

`helm upgrade --install` is idempotent: installs if absent, upgrades if present, rolls back on failure.

### Nexus Repository OSS

| Aspect | Strategy |
|---|---|
| **Idempotency** | Proxy repos cache on first fetch — subsequent fetches hit cache |
| **Free plan** | Apache 2.0 (free). All format types included. |
| **Limitations vs Pro** | No HA clustering, no staging/promotion, no S3 blob store |
| **Cleanup** | Blob store cleanup policy: remove unused after 30 days |

**Resilience:** Any package fetched once is cached permanently (until cleanup policy). Upstream removal, deprecation, or outage has no impact on builds.

### Kaniko (Container Builds)

| Aspect | Strategy |
|---|---|
| **Idempotency** | Docker layer caching — unchanged layers are not rebuilt |
| **Free plan** | Apache 2.0 (free) |
| **Security** | Rootless, daemonless. `RUN --mount=type=secret` for build-time secrets. Never `--build-arg` for secrets. |

---

## 3. Secrets Management

### HashiCorp Vault OSS

| Aspect | Strategy |
|---|---|
| **Idempotency** | `vault kv put` overwrites — safe to repeat. Versioned KV v2 keeps history. |
| **Free plan** | MPL 2.0 (free). No namespaces, no HSM, no replication. |
| **Limitations vs Enterprise** | No namespace isolation (use path-based separation), no sentinel policies, no HSM auto-unseal |
| **Backup** | `vault operator raft snapshot` → MinIO (daily, Ansible-scheduled) |
| **NetBox integration** | Vault paths follow service slugs: `secret/{scope}/{service}` |

**Secret path convention:**
```
secret/platform/gitlab     → GitLab secrets
secret/platform/netbox     → NetBox API tokens
secret/infra/arista        → Arista switch credentials
secret/mod/voip/kamailio   → Kamailio DB password
```

**Idempotent secret injection:**
```yaml
# Ansible: read from Vault, inject into service config
- name: Get GitLab DB password
  set_fact:
    gitlab_db_password: "{{ lookup('hashi_vault', 'secret/data/platform/gitlab:db_password') }}"
```

### Vaultwarden

| Aspect | Strategy |
|---|---|
| **Free plan** | AGPL v3 (free, self-hosted) |
| **Purpose** | Human credentials only. Bitwarden-compatible. |
| **Backup** | SQLite DB + attachments → MinIO (daily) |

---

## 4. Identity and Access

### Authentik

| Aspect | Strategy |
|---|---|
| **Idempotency** | Flows, providers, and applications are declarative via Authentik API or Terraform provider |
| **Free plan** | MIT license (free, self-hosted). All features included. |
| **NetBox integration** | Authentik provides OIDC for NetBox via proxy auth or plugin |

### Teleport CE

| Aspect | Strategy |
|---|---|
| **Idempotency** | Certificate-based access — no `authorized_keys` file drift |
| **Free plan** | Apache 2.0 (free). Session recording, MFA, OIDC included. |
| **Limitations vs Enterprise** | No FedRAMP, no hardware key support, limited access requests workflow |
| **Session recording** | 90-day retention → MinIO. Monitor storage via Prometheus. |

---

## 5. Monitoring and Observability

### Prometheus + Alertmanager

| Aspect | Strategy |
|---|---|
| **Idempotency** | Scrape config is declarative. Reloading config re-applies targets. |
| **Free plan** | Apache 2.0 (free) |
| **NetBox integration** | Each service's `prometheus_job` custom field maps to a scrape target |

### Grafana

| Aspect | Strategy |
|---|---|
| **Idempotency** | Dashboards as code (JSON/Jsonnet) provisioned via `grafana-provisioning/` directory |
| **Free plan** | AGPL v3 (free, self-hosted). All features. |
| **Dashboard tags** | `service:{name}`, `env:{env}`, `scope:{scope}` — matches NetBox |

### Loki

| Aspect | Strategy |
|---|---|
| **Idempotency** | Log ingestion is append-only. Config is declarative. |
| **Free plan** | AGPL v3 (free, self-hosted) |
| **Storage** | MinIO S3 backend (retention policy: 90 days default) |

### Zabbix

| Aspect | Strategy |
|---|---|
| **Idempotency** | Zabbix API for host/template management. `zabbix_host` Ansible module is idempotent. |
| **Free plan** | GPL v2 (free) |
| **Purpose** | SNMP/agentd for network devices and legacy systems (supplements Prometheus) |

---

## 6. Security and Compliance

### Wazuh

| Aspect | Strategy |
|---|---|
| **Idempotency** | Agent registration is idempotent (re-registration updates existing agent) |
| **Free plan** | GPL v2 (free, self-hosted). All SIEM/IDS/compliance features. |
| **Dashboards** | ISO 27001, NIS1/NIS2 compliance views built-in |

### Scanning Pipeline (CI)

All scanners are idempotent — re-running produces fresh results:

| Tool | Scope | Free plan |
|---|---|---|
| Trivy | Container + IaC + secrets + SBOM | Apache 2.0 (free) |
| Checkov | IaC security (Terraform, Ansible, Docker, K8S) | Apache 2.0 (free) |
| Semgrep | SAST — static code analysis | LGPL (free, OSS rules) |
| OWASP Dependency-Check | Dependency CVE scan | Apache 2.0 (free) |
| Gitleaks | Git secret scanning | MIT (free) |
| OpenVAS / Greenbone | Network vulnerability scanner | GPL v2 (free, self-hosted) |
| Prowler | K8S CIS benchmark + NIS2 | Apache 2.0 (free) |

**All findings feed into DefectDojo** (BSD, free) — single vulnerability dashboard.

### CISO Assistant + Eramba

| Tool | Purpose | Free plan |
|---|---|---|
| CISO Assistant | GRC governance — risk register, compliance evidence, audit trails | AGPL v3 (free) |
| Eramba Community | ISO 27001 program management | AGPL v3 (free) |

---

## 7. Storage

### PostgreSQL (Patroni)

| Aspect | Strategy |
|---|---|
| **Idempotency** | Schema migrations via versioned migration tools (Flyway, Alembic, or built-in). Replay-safe. |
| **Free plan** | PostgreSQL License (free). Patroni is Apache 2.0 (free). |
| **HA** | Single instance for PoC → Patroni cluster for production |
| **Backup** | `pg_dump` daily → MinIO. WAL archiving for point-in-time recovery. |

### MinIO

| Aspect | Strategy |
|---|---|
| **Idempotency** | S3 PUT is idempotent — same key overwrites same object |
| **Free plan** | AGPL v3 (free, self-hosted) |
| **Purpose** | S3-compatible storage for Loki, backups, GitLab artifacts, recordings |

### Redis (Sentinel)

| Aspect | Strategy |
|---|---|
| **Idempotency** | `SET` is idempotent. Sentinel provides HA failover. |
| **Free plan** | BSD-3 (free) |
| **HA** | Single instance for PoC → Sentinel for production |

---

## 8. Networking

### pfSense CE

| Aspect | Strategy |
|---|---|
| **Idempotency** | pfSense API + Ansible module — declarative rule management |
| **Free plan** | Apache 2.0 (free). pfBlockerNG is free add-on. |
| **Backup** | XML config backup → MinIO (daily, Ansible-scheduled) |

### Traefik v3

| Aspect | Strategy |
|---|---|
| **Idempotency** | Declarative config via labels (Docker) or IngressRoute CRDs (K8S) |
| **Free plan** | MIT (free). All features. |
| **TLS** | step-ca for `*.by-systems.internal`, Cloudflare DNS-01 for `*.by-systems.be` |

### Bind 9

| Aspect | Strategy |
|---|---|
| **Idempotency** | Zone files are declarative. Serial number increment on change. |
| **Free plan** | MPL 2.0 (free) |
| **NetBox integration** | DNS records can be generated from NetBox IP assignments |

---

## 9. Documentation

### Markdown + PlantUML + Sphinx

| Aspect | Strategy |
|---|---|
| **Idempotency** | Docs are declarative text files — rebuilding always produces same output |
| **Free plan** | All tools are free/open source |
| **Diagrams** | PlantUML/Mermaid source → KROKI renders → PNG export |
| **Build** | Sphinx pipeline: `.md` → HTML / PDF / DOCX on `main` merge |

---

## 10. GitOps Principles

All platform configuration follows GitOps:

1. **Git is the source of truth for configuration** — all config is versioned in Git
2. **NetBox is the source of truth for infrastructure state** — devices, IPs, services
3. **Vault is the source of truth for secrets** — never in Git, never in environment variables
4. **Changes are made via merge requests** — no manual edits on production systems
5. **Automation converges to declared state** — Ansible, Terraform, Helm all converge
6. **Drift is detected and corrected** — `terraform plan`, `ansible --check`, Prometheus alerts

### Free Plan Summary

| Category | Tool | License | Limitation vs Paid |
|---|---|---|---|
| Compute | Proxmox VE | AGPL v3 | No enterprise support |
| IaC | Terraform CLI | MPL 2.0 | No Terraform Cloud (use MinIO for state) |
| IaC | Ansible | GPL v3 | No AAP/Tower (use Semaphore) |
| CI/CD | GitLab CE | MIT | No advanced RBAC, no DORA metrics |
| Proxy | Nexus OSS | Apache 2.0 | No HA, no staging/promotion |
| Identity | Authentik | MIT | All features included |
| Secrets | Vault OSS | MPL 2.0 | No namespaces, no HSM |
| Bastion | Teleport CE | Apache 2.0 | No FedRAMP, limited access requests |
| IPAM/CMDB | NetBox | Apache 2.0 | No commercial support |
| Monitoring | Prometheus/Grafana/Loki | Apache/AGPL | All features included |
| SIEM | Wazuh | GPL v2 | All features included |
| GRC | CISO Assistant | AGPL v3 | All features included |
| Storage | PostgreSQL + MinIO + Redis | Various OSS | Self-managed HA |
| K8S | k3s | Apache 2.0 | Lightweight; evaluate RKE2 at scale |
