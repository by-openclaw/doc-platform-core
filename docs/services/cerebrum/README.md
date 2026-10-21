# Cerebrum on the platform — service README (draft, 2026-10-21)

**Status:** draft for the owner; grows with the service ADR and the roles. Facts come from the lab inspection and the repository review (`docs/reviews/2026-10-21-*.md`); rows marked *to confirm* wait for the vendor.

## 1. What Cerebrum is here

EVS Cerebrum, the broadcast control and monitoring system: one **server** (desktop application in a service session, with its Watchdog), a **Device Service** (Service Manager, a Windows service that talks to devices on the server's behalf), **clients** (desktop client, web client, Designer seats), **hardware panels**, an optional **Federation** service (multi-site), an optional **SDN** orchestrator, and the **VIA License Manager** that licenses EVS products (Cerebrum, Neuron, …).

## 2. Hosts and scopes

| Scope | Host | Playbook | Role |
|---|---|---|---|
| database | vm-winsql-01 | `cerebrum-db.yml` | `cerebrum_db` |
| server (+ Watchdog, Designer seat) | vm-winapp-01 | `cerebrum-server.yml` | `cerebrum_server` |
| device service (Service Manager) | vm-winapp-02 | `cerebrum-device-service.yml` | `cerebrum_device_service` |
| licence manager | vm-winapp-02 (to decide) | `cerebrum-license.yml` | `cerebrum_license` |
| NMOS registry | vm-winapp-02 | `cerebrum-nmos.yml` | `cerebrum_nmos` |
| clients | Win11 Pro workstations | `cerebrum-client.yml` | `cerebrum_client` |
| configuration export/import | playbook only | `cerebrum-db-config.yml` | — |
| operations (start, stop, version report) | playbooks only | `cerebrum-ops.yml` | — |

Every role: pinned installer with checksum from the platform object store, settings as data, secrets in Vault, firewall rules per service and port, logging to Loki, a liveness probe, a backup line, and a CISO control map (`docs/adr/security/0002`).

## 3. Owner's decisions (2026-10-21)

| # | Topic | Decision |
|---|---|---|
| 1 | UDP 2071 / 2072 | ACP1 on 2071, ACP2 on 2072 (Axon Control Protocol). UDP 2000 seen listening on the lab: *still to confirm with the vendor* |
| 2 | Licensing | VIA License Manager is the licensing service for EVS products (Cerebrum, Neuron, …): one instance on the platform |
| 3 | Unattended run | needed: service session account, auto-logon, Watchdog, start-at-logon tasks; CIS exception on that host |
| 4 | Database | SQL Server with an enhanced install (static port, no Browser, no telemetry, backups) **but the vendor's specification for the Cortex database, mirroring and backup is kept** |
| 5 | Clients | desktop client on the workstations; web client available; one hardware panel to install later |
| 6 | Logging | Cerebrum supports **syslog and SNMP, syslog by default**. The syslog **format is not standard** (fields in the wrong columns): capture a sample from the lab, write the parser for it, then compare with SNMP before choosing the alerting source |
| 7 | SDN | probably not needed; not deployed unless a device requires it |
| 8 | NMOS discovery | all three: mDNS works on the local test platform without a reflector; every device that offers this connection is tested with it; unicast DNS-SD and manual as the other paths (see §7) |
| 9 | HA | the vendor's A/B pairing is Cerebrum's own mechanism, not hypervisor clustering; the Watchdog / Witness services may no longer be used in 2.9 — *to check in the technical documentation*; see §8 |
| 10 | Upgrades | yes, after the configuration files are exported and templated (done 2026-10-21: 22 files in the quarantine export) |
| 11 | Windows Server licence | parked until the WS2025 template is tuned and the hypervisor cluster exists; the three guests stay pinned |
| 12 | Tooling | everything by Ansible: installers, browser, Wireshark, tools, VS Code with extensions (Python, XML, JSON, YAML, Ansible, SSH keys, Vault) — the `infra/0011` profiles |

## 4. Ports and protocols (lab 2.9.3, to confirm with the vendor's list)

| Component | Port | Protocol | Purpose |
|---|---|---|---|
| server | 40001, 40002 | TCP + UDP | server protocol: clients, panels |
| server | 40009 | TCP | link to the Device Service |
| server | 9081 / 9080 | TCP | web client, RCP over HTTP |
| server | 162 | UDP | SNMP trap receiver |
| server | 1161 | UDP | Cerebrum's SNMP agent |
| server | 2071, 2072 | UDP | ACP1, ACP2 (Axon Control Protocol) |
| server | 2000 | UDP | listening on the lab, *to confirm* |
| server | 40003, 40030 | TCP | Federation (from the repository) |
| server | 40007, 40051 | TCP | Mediaconf and Exchanger XML APIs (from the repository) |
| server | 8080–8085, 8090–8095 | TCP | FCR and SAMIX APIs (from the repository) |
| Device Service | 40009, 31150–31155, 62010 | TCP | management port, XS via Linx, video mixer (from the repository) |
| NMOS registry | 8080 | TCP | `x-nmos/registration`, `x-nmos/query`; mDNS 5353 UDP |
| licence manager | 8558 | TCP | API and UI, TLS 1.2 |
| database | static port to set | TCP | from the server host only |
| SDN (if ever) | 5000, 5002 | TCP | IS-06 API, gRPC |

## 5. Device families the vendor library ships (lab 2.9.3)

AMU, Antenna Controller, Assignment Engine, Audio Console, Audio Processor, Automation, AV Devices, Axon, Cameras, CiaB, Cloud Routing Orchestrator, Cloud Workflow, Compression Platform, Conference Call System, Dataprobe, Delay Line Units, DSOM, Ember, EVS (BRIDGE, SHUFFLE), GPIO, Graphics Engine, GV Densité, Intercom, IP Media Gateway, IP Network Orchestrator, IR Devices, IRD, KVM Switch, LED Wall Controller, Lighting Console, Management Systems, Media Player, Media Server, MediorNet, Miranda iControl, Monitors, Multiviewers, NMOS, Panels, Projectors, Protocols, Pseudo, RF Devices, RollCall, Ross OpenGear, Routers, Schedule Management, Scoreboard Protocols, Servers, Set Top Box, Simulated Devices, Studio Timers, Subtitling, Switch, Switchers, TBU, Test Instruments, UMDs, Video Processing, VoIP, VTECS RPC, VTR.

**Devices and features we use:** *to fill by the owner* — the devices on site, and the Cerebrum features in use (routing, tally and UMD, panels, NMOS, SNMP monitoring, macros, scheduling, web client, federation).

## 6. Where configuration lives (templated by the roles)

| File | Content | Source of truth here |
|---|---|---|
| `Cerebrum.ini` | SQL connection, up to four bind addresses, Cortex link, licence | Jinja2 template from inventory data + Vault |
| `Configurations\*.ccf` | the site design; nightly auto-backup at 02:30, `LastShutdown.ccf` | the owner's design, backed up by the guest backup and exported by `cerebrum-db-config` |
| `Global Data Files\*.XML`, `LOCAL DATA FILES`, `Data Files\*` | clients, users, menus, routing modes, virtual routers | exported 2026-10-21; templated where static |
| `Projects`, `Forms`, `GenericDevices`, `Python Scripts` | panels, UI, device library, device drivers | vendor-shipped; owner additions versioned in git |
| SDN `etc\*.json`, `prometheus.yml` | orchestrator settings | only if deployed |
| licence manager `Config\*.conf`, certificate | TLS, LDAP (Authentik possible) | role data + Vault |

## 7. Discovery (NMOS, mDNS, DNS-SD)

- **mDNS** is link-local multicast: it reaches the VLAN it is sent on. On the local test platform it works without any reflector (owner, 2026-10-21); every device that offers this connection is tested with it. Across VLANs it would need a reflector on the firewall, which is a decision per device, not a default.
- **Unicast DNS-SD** is ordinary DNS (`_nmos-register._tcp`, `_nmos-query._tcp` SRV/TXT/PTR records) served by the platform resolver: it works across VLANs and is the preferred path. *To confirm:* which EVS devices and Cerebrum 2.9.3 support unicast DNS-SD.
- **Manual** registry address on the device is the fallback.

## 8. Resilience and HA, two different layers

| Layer | What it protects against | How | When |
|---|---|---|---|
| hypervisor cluster (the DL380 plan, Ceph, live migration) | a host failure, maintenance without downtime | the guest moves or restarts on another node; the application restarts | the cluster program; the guests stay pinned until then |
| application HA (the vendor's A/B pairing — Cerebrum's own mechanism, not hypervisor clustering) | an application or OS failure | two servers A/B with SQL mirroring (SQL Standard); the Watchdog / Witness services of older releases may no longer apply in 2.9 — *to check in the technical documentation* | when licensed and after the first site runs |

A cluster does not make the application highly available; it makes the host disposable. Both are wanted eventually; the cluster comes first because it serves every service.

## 9. Logging and monitoring

| Source | Today | On the platform |
|---|---|---|
| syslog from Cerebrum (default output; **non-standard format**, fields in the wrong columns) | not configured on the lab | Alloy's syslog listener receives it; a parser written from a captured sample (the lab points its syslog at the platform's listener for the capture); labels by severity → Loki |
| audit trail `ErrorLogs\AuditTrail\*.log` (TSV, severity column, 10 MB rotation) | file only | Alloy tails it → Loki as the second source until the syslog parser is proven; alerts on crash dumps |
| SNMP | Cerebrum's agent on 1161 and traps out | Prometheus `snmp_exporter` with the shipped `AppMIBs` for health; a trap receiver later if the comparison shows events the audit trail lacks |
| SQL Server errorlog, licence manager, Windows events | event log | Alloy channels already in place |
| liveness | — | TCP probe on 40001, the web client on 9081, the licence manager on 8558 |

## 10. Runbook (to grow)

| Action | How |
|---|---|
| start / stop the server session applications | `cerebrum-ops.yml -e action=start|stop` |
| version report of every host | `cerebrum-ops.yml -e action=versions` |
| export / import the `_Config` table and the data files | `cerebrum-db-config.yml` |
| restore the site design | the latest `.ccf` from the guest backup or the export |
| restore the database | SQL-native backup from the NAS (vendor procedure for Cortex) |
| licence manager down | clients keep running on the grace period (*to confirm*); restore the guest |
| upgrade | `cerebrum-server.yml -e cerebrum_version=<new>` after the export, vendor switches `/COMPONENTS`, `/InstallSQL=Skip`, `/InstallSentinel=Skip` |

## 11. CISO map per service (to complete in the service ADR)

Each Windows service gets its own row: account it runs as, ports, firewall rule, TLS, logging, backup, update path, NIS2 and ISO 27001 controls — server session, Device Service, Federation, licence manager, NMOS registry, SQL instance, SDN if any.
