# services/0007 — Provisioning Orchestrator

**Status:** Draft
**Date:** 2026-05-23
**Scope:** End-to-end provisioning contract for a new VM / LXC / VPS — the cross-system orchestration between NetBox, Terraform, Proxmox, OPNsense, DNS, Ansible, and observability. Does not specify a single tool; defines the steps, ownership, and ordering.
**Related:** `services/0001-opnsense`, `services/0003-netbox-cmdb`, `services/0006-firewall-services`, `infra/0003-terraform-standard`, `infra/0004-network-architecture`, `infra/0006-logging`, `security/0001-secret-storage`

---

## Context

Provisioning a new workload (LXC, VM, future VPS) currently means **the operator does the right thing in the right order across multiple systems** — register in NetBox, allocate IP, create in Proxmox via Terraform, tag the right VLAN, add a DHCP reservation or static lease on OPNsense, register DNS in Cloudflare/Unbound, run Ansible to configure the OS, ensure logs land in Loki. **A single missed step results in a workload that exists in one system but not another (silent drift).**

This ADR fixes the orchestration contract: an ordered chain of steps with explicit responsibility per step, and a graceful-fallback rule for the period before every dependency is online (NetBox is not yet deployed; DNS is partially manual; etc.).

## Decision

### 1. Provisioning chain — fixed order

For every new workload (LXC, VM, future VPS), the following 8 steps execute in this order. Each step is idempotent; failure halts the chain and rolls back where reversible.

| # | Step | System | Owner (today) | Owner (target) |
|---|---|---|---|---|
| 1 | **Reserve identity** (name, env, role, owner) | NetBox | Operator | NetBox API |
| 2 | **Allocate IP** (IPAM, VLAN, gateway, MAC if needed) | NetBox IPAM | Operator | NetBox API |
| 3 | **Create resource shell** (CPU/RAM/disk, NIC, VLAN tag) | Proxmox via Terraform | Terraform module | Same |
| 4 | **Register at firewall** (DHCP static lease if needed, alias membership if part of a security group) | OPNsense via lib-opnsense | Pass 1 script | Ansible role |
| 5 | **Register DNS** (A + AAAA — internal Unbound; public via Cloudflare if exposed) | OPNsense Unbound + Cloudflare | Pass 1 script | Ansible role |
| 6 | **Apply OS baseline** (users, SSH keys, packages, hardening) | Ansible playbook | Operator | CI on merge |
| 7 | **Apply role config** (service-specific Ansible role) | Ansible playbook | Operator | CI on merge |
| 8 | **Verify observability** (host appears in Loki, Prometheus, Grafana dashboards) | Loki/Prom/Grafana API | Operator | Health-check job |

### 2. Graceful fallback — degraded mode

The chain must work even if some target dependencies are not yet deployed. Each step declares a fallback:

| Step | Required dependency | Fallback when missing |
|---|---|---|
| 1 | NetBox | YAML inventory file in `ansible-platform/inventories/{env}/hosts.yml` is the registry. Mark `netbox: pending`. |
| 2 | NetBox IPAM | Static IP assigned in the YAML inventory + `_ip_source: manual` annotation. |
| 3 | Terraform | Direct Proxmox API call from orchestrator script. Mark `terraform: bypassed` in audit log. |
| 4 | lib-opnsense / FW | Skip; emit warning. Workload provisioned but cannot use IBR or rely on static DHCP. |
| 5 | Cloudflare/Unbound | Skip DNS registration; record name + IP in inventory. |
| 6–7 | Ansible | Skip; workload boots with cloud-init defaults only. |
| 8 | Loki/Prom | Skip verification; emit warning. |

**Every fallback emits an audit record** to RAID + (when available) Loki with label `service=orchestrator action=fallback step=N`.

### 3. Single orchestrator entrypoint

A single CLI / API entrypoint executes the chain:

```
orchestrator provision <workload-type> <spec.yaml>
   --env <test|prod>
   --dry-run | --apply
   --skip-step N         # explicit skip with reason
```

Implementation: an **Ansible master playbook** (`ansible-platform/playbooks/orchestrate.yml`, aka `site.yml`) — the platform is Ansible-first (config management = Ansible only), so the orchestrator stays in Ansible rather than a parallel Python control plane. It:
- Loads the workload spec (YAML — name, role, env, VLAN, size, disk, etc.) as host/group vars
- Walks the 8 steps in order, **one play/role import per step**
- Drives each system through its Ansible module or a **Python leaf library** (`lib-opnsense` for FW/DNS, `lib-synology` for NAS, `community.general.terraform` for the resource shell) — the libraries are leaf tools called *from* Ansible, **not** a separate orchestration engine
- Fallback (degraded mode) = `block`/`rescue` + `when: <dependency_available>`; **`--check` is the dry-run**
- Logs every step + outcome to Loki (when available) + stdout + a RAID line
- Fails fast: any un-rescued step halts the run (non-zero)

Target trigger: a GitLab pipeline on an MR to `ansible-platform/inventories/{env}/hosts.yml` runs this playbook — the MR diff is the spec; the pipeline invokes the orchestrator.

### 4. Spec format

A workload spec is the **single source of truth** for what the workload is. All systems read this spec; none of them invent fields.

```yaml
# Minimal LXC spec
kind: lxc
name: lxc-test-mgmt
env: test
role: test-client
owner: yboujraf
network:
  vlan: 2010                     # ADR infra/0004 §4
  zone: mgmt
  ip: 10.11.1.100                # static; allocated from NetBox once live
  gateway: 10.11.1.1
resources:
  cpu: 1
  ram_mb: 512
  disk_gb: 4
  template: debian-13-standard
fw:
  ibr_initial: false             # not in host_internet_request at boot
dns:
  zone: test.by-research.be      # naming/0001 §9
  public: false
ansible:
  # step 6 — host baseline, applied to EVERY host, in identity/0004 §9 order:
  baseline: [user-mgmt, sshd-hardening, banner, hardening, ca-trust]
  # step 7 — service-specific roles (empty for a bare test client):
  roles: []
observability:
  loki: true
  prometheus_exporter: node
```

The spec is the input. NetBox / Terraform / OPNsense / Ansible all derive their state from it. **No system invents fields not in the spec.**

### 5. Idempotency and replay

- The orchestrator is **idempotent**: running it twice for the same spec must produce no changes the second run.
- Each step calls a manager / module that itself implements `ensure()` semantics.
- A failed run can be replayed after the failure cause is fixed; partial state from the first run does not block the replay.
- `--dry-run` walks the chain, computing the diff per step, without making changes. Used in CI for MR validation.

### 6. Audit and observability

Every orchestrator run emits a structured log entry per step:

```
service=orchestrator
run_id=<uuid>
workload=<name>
env=<env>
step=<N>
step_name=<reserve_identity|allocate_ip|create_shell|register_fw|register_dns|os_baseline|role_config|verify_obs>
outcome=ok|fallback|skipped|error
duration_ms=<int>
ref_adr=<scoped path>
detail=<short string>
```

A run is "successful" only if all 8 steps return `ok` or an explicitly justified `skipped`. `fallback` counts as success but is flagged in the daily orchestrator-health report.

### 7. Decommissioning — the reverse chain

Decommissioning runs the same 8 steps in reverse, each as `state=absent`:
8. Remove from observability
7. Remove role config
6. Remove OS baseline
5. Remove DNS records
4. Remove FW registrations / alias memberships
3. Destroy Proxmox resource (Terraform `destroy` or direct API)
2. Release IP back to IPAM
1. Mark identity as decommissioned in NetBox (never deleted — history preserved)

### 8. Cross-OS scope

The chain is the same for Linux VMs, LXCs, FreeBSD (OPNsense), Windows VMs (future). Per-OS specifics live in Step 6 (Ansible role); the orchestrator does not branch on OS at steps 1–5 or 7–8.

## Consequences

- **Provisioning is reproducible** — every workload comes from a spec, walked by one orchestrator.
- **Drift between systems is detectable** — the spec is the truth; differences from spec are drift events.
- **Graceful degradation in early platform** — NetBox missing does not block LXC creation; Cloudflare missing does not block FW work. Every fallback is audited.
- **Single audit trail** per provisioning run via `run_id`.
- **Decommissioning is not an afterthought** — same orchestrator, reverse chain, same audit.
- **CI-driven future** — once `orchestrate.py` is stable and GitLab is up, MRs to inventory become the provisioning interface; no operator runs anything by hand.

## Revision triggers

Revise when:
- A 9th cross-system concern appears (e.g. secrets injection at provisioning time)
- NetBox is replaced by another CMDB
- Terraform is replaced by Crossplane / Pulumi
- A separate orchestration engine is adopted (Argo, Temporal)
- Multi-cluster provisioning needs add cross-cluster steps
- The spec format is breaking-changed (versioned spec required)

## CISO mapping

| Framework | Controls covered |
|---|---|
| ISO 27001:2022 | A.5.9 (asset inventory — NetBox-derived), A.8.9 (configuration management — spec-driven), A.8.16 (monitoring — audit per step), A.8.32 (change management — every provision/decom audited) |
| NIS2 | Art. 21(2)(c) (business continuity — reproducible provisioning), Art. 21(2)(e) (network and information systems security — FW and DNS in the same chain as VM creation) |
