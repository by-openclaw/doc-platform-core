# Platform Status

> **Last updated:** 2026-10-03
> **Maintained by:** Rune — update after any release, layer change, or RAID update
> **Reading order:** `status.md` → `roadmap.md` → [`docs/adr/README.md`](adr/README.md) → relevant repo `CLAUDE.md`

This page tracks platform **deployment and delivery state**. The per-service detail (host, exposure, SSO, pin, backup, monitoring, audit record) is generated in `ansible-platform/docs/register.md` — this page does not copy it.

---

## Layer progress

Per [`infra/0002-platform-charter §Layer model`](adr/infra/0002-platform-charter.md).

| Layer | Name | Status | Notes |
|---|---|---|---|
| 0 | Standards & Templates | ✅ **Complete** | Scoped ADRs under `docs/adr/`, `OPERATING-STANDARD.md`. Open proposals: PRs #60, #65–#69. |
| 1 | Proxmox Base | ✅ **Operational** | The hypervisor is built from the repo (`ansible-platform/roles/pve_host`, release pin + ladder); every guest carries the hardening baseline. **Open:** one pool disk failed on 2026-10-03 (pool degraded, kernel reboot held until it is replaced). |
| 2 | Vault + step-ca | ✅ **Deployed** (single instance) | Vault 2.1.1 (AppRole deploy model, boot-time unseal by the warden), step-ca 0.30.2, platform CA trusted on every guest. **Open:** HA (charter: 2 instances), offline custody of the unseal kit and of the root CA key. |
| 3 | Identity (Authentik) | ✅ **Deployed** (single instance) | People and groups from one source (`people.yml`), OIDC / forwardAuth on every service, MFA enrolment enforced. **Open:** HA (proposal #66). |
| 4 | Storage | ✅ **Deployed** | Synology NAS (NFS) + SeaweedFS as the S3 store (replaced MinIO per [`services/0009`](adr/services/0009-object-storage.md)); backups: PBS on S3, NAS copy with retention, off-site replication. |
| 5 | Platform Services | ✅ **32 services deployed** | All on the service contract; the ADR compliance walk closed on 2026-10-03 (one audit record per service). |

---

## What the ADR tool list still misses

Against the original stack decision (flat ADR-0001, 2026-03-25) and the current scoped ADRs. Owner decisions of 2026-10-03 are marked.

| Listed tool | State | Decision |
|---|---|---|
| Neo4J graph CMDB | not deployed | waits for the NetBox population (owner: wait) |
| PostgreSQL Patroni cluster + Redis Sentinel | single instance each | to build ([`services/0004`](adr/services/0004-database-strategy.md)) — after the pool disk is replaced |
| Vault HA, Authentik HA | single instance each | to build — same condition |
| Headlamp + k9s | not deployed | to build |
| Nexus Repository OSS | not deployed | **dropped by the owner** (Community Edition is capped); GitLab CE + Harbor + Verdaccio are the repositories |
| Sphinx | not deployed | **replaced by the owner**: Markdown + PlantUML + Kroki |
| Packer | not used | proposal: drop (guests come from Debian cloud images through Terraform) |
| Checkov, Gitleaks, OWASP Dependency-Check | not used | proposal: drop (the CI gate is Trivy: vulnerabilities, secrets, misconfiguration) |
| DefectDojo, OpenVAS, Falco, OpenSCAP, Prowler | not deployed | proposal: not adopted (Wazuh covers host vulnerabilities and CIS checks) |
| WireGuard on the firewall | not configured | NetBird is the VPN (WireGuard is its data plane) |
| Teleport, Guacamole | not deployed | replaced by JumpServer CE ([`security/0003 §8`](adr/security/0003-hardening.md)) |
| Pi-hole, MinIO, Zabbix | not deployed | replaced by AdGuard and SeaweedFS; Zabbix excluded by [`infra/0007`](adr/infra/0007-monitoring.md) |

New modules asked by the owner on 2026-10-03, not in any ADR yet: **IoT** (Zigbee coordinator, MQTT broker, sensor values to Prometheus/Grafana), **CCTV** (stream recorder with history, NFS + S3 storage), **RADIUS** (network access for IoT, switches, Wi-Fi access points).

---

## Per-repo status

| Repo | Role | State |
|---|---|---|
| `doc-platform-core` | Platform documentation | ADRs current; this page and `roadmap.md` refreshed 2026-10-03 |
| `ansible-platform` | Roles and playbooks for the whole platform | 32 services on the contract, one role per concern, register generated from the catalog |
| `infra-terraform-proxmox` | Guests (VM/LXC) and the firewall seed | prod environment live; firewall built from the seed |
| `lib-opnsense` / `ansible-opnsense` | Firewall library and collection | firewall catalog applied through them; API verified on OPNsense 26.7.5 (probe data for 26.7.5 still to record from a non-production firewall) |
| `lib-synology-dsm` | NAS library | unchanged |
| `platform-setup` | Runbooks | unchanged |

Source hosting: the repositories are still on GitHub; GitLab CE is deployed and the migration of [`git/0002`](adr/git/0002-platform-strategy.md) has not started.

---

## Open items

| # | Item | Owner |
|---|---|---|
| 1 | Replace the failed pool disk on the hypervisor, then resilver; then the held kernel reboot | owner (hardware), then Rune |
| 2 | HA data layer and HA for Vault / Authentik | Rune, after item 1 |
| 3 | ADR proposals #60, #65–#69 | owner (merge) |
| 4 | NetBox population | waiting (owner) |
| 5 | IoT, CCTV, RADIUS modules | design proposed; build needs the devices on their VLANs |
| 6 | Debian 13 for the mail VM and the CI-runner VM (in-place upgrade playbook exists) | Rune, after item 1 |
| 7 | Custody: Vault unseal kit, step-ca root key offline; mail reverse DNS at the ISPs; revoke the retired relay key | owner |

---

## Recent updates

| Date | What changed |
|---|---|
| 2026-10-03 | ADR compliance walk closed: 32 of 32 services audited, fixed, applied twice, merged. Maintenance window: PostgreSQL data checksums, Vault 2.1.1, Debian 13 on the k3s and resolver VMs (the mail and CI-runner VMs remain on Debian 12), firewall 26.7.5, object-storage quota, NAS backup retention. Monitoring reduced to the liveness set (owner direction). A pool disk failed during the window. |
| 2026-10-01 | Service contract complete: every service containerised and on the contract (32 of 32). |
| 2026-09 | Platform build-out: firewall rebuilt from the seed on 26.7, GitLab CE, Harbor, Verdaccio, JumpServer, Wazuh, CISO Assistant, SeaweedFS, monitoring and alerting, backups. |
| 2026-04-14 | Scoped ADR refactor complete (flat ADRs archived). |

---

## How to update

After any release or layer change:

1. Update §Layer progress and §Open items
2. Update the "missing" table when a tool is deployed, dropped or replaced
3. Add a row to §Recent updates (newest first)
4. Commit: `docs: update platform status to YYYY-MM-DD`
