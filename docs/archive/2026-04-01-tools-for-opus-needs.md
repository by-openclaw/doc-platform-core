# Tools in platform-setup — For Opus NEEDS.md Triage

> **From:** Rune
> **To:** Opus
> **Date:** 2026-04-01
> **Request:** For each tool below that doesn't already have a NEEDS.md entry, create one using the standard format.
> **Action:** Opus proposes entries → @yboujraf approves → Rune adds to NEEDS.md

---

## Currently tracked tools in platform-setup/tools/

| Tool | Directory | Already in NEEDS.md? | Notes |
|---|---|---|---|
| authentik | tools/authentik | Yes (Layer 3 — Identity) | OIDC/SSO gateway |
| gitlab | tools/gitlab | Yes (Layer 3 — VCS) | GitLab CE self-hosted |
| gitlab-runner | tools/gitlab-runner | Likely | CI runners |
| grafana | tools/grafana | Yes (Layer 5 — Observability) | Dashboards |
| loki | tools/loki | Yes (Layer 5 — Observability) | Log aggregation |
| netbox | tools/netbox | Yes (H9) | IPAM/DCIM |
| network | tools/network | Partial | Arista + pfSense config |
| nexus | tools/nexus | Unknown | Package registry (Maven/npm/pip) |
| openclaw | tools/openclaw | N/A | Agent runtime |
| pfsense | tools/pfsense | Partial | Firewall/router |
| pihole | tools/pihole | Unknown | DNS sinkhole |
| prometheus | tools/prometheus | Yes (Layer 5 — Observability) | Metrics |
| proxmox | tools/proxmox | Yes (Layer 1) | Hypervisor |
| step-ca | tools/step-ca | ❌ No — deferred | Internal CA (*.org.internal) — keep skeleton, defer work |
| synology | tools/synology | Yes (multiple entries) | NAS |
| traefik | tools/traefik | Unknown | Reverse proxy / ingress |
| unifi | tools/unifi | Unknown | Network management |
| vault | tools/vault | Yes (Layer 2) | Secrets management |
| wireguard | tools/wireguard | Unknown | VPN |
| zabbix | tools/zabbix | Unknown | Network monitoring |

---

## Tools with unknown/missing NEEDS.md status — Opus to assess

These tools exist in platform-setup but may not have explicit NEEDS.md entries:

- **nexus** — Package registry. Is this planned? What phase?
- **pihole** — DNS sinkhole. Standalone or replacing pfSense DNS?
- **traefik** — Reverse proxy. Is this the ingress strategy (vs nginx, Caddy)?
- **unifi** — Network management. Is Unifi hardware in the platform?
- **wireguard** — VPN. Is this replacing pfSense VPN or supplementing it?
- **zabbix** — Network monitoring. Is this alongside or replacing Prometheus?

---

## Request to Opus

For each "Unknown" tool above:
1. Check platform-setup issues and NEEDS.md for any existing reference
2. If no entry exists: propose a NEEDS.md entry (priority, phase, brief description)
3. Flag any that seem out of scope or should be removed from platform-setup/tools/

@yboujraf approves proposals. Rune adds to NEEDS.md after approval.

---

*Filed by Rune. Opus reviews. @yboujraf decides.*
