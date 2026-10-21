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

## 12. Configuration reference for the vendor discussion (lab 2.9.3, 2026-10-21)

Cerebrum keeps four layers of configuration. The table carries one row per item, what the lab holds, and the question or the control it needs.

### 12.1 `Cerebrum.ini` — the bootstrap file (one file, written at install, read at start)

| # | Key | Lab value | Meaning | Control / question for the vendor |
|---|---|---|---|---|
| 1 | `CORTEX:DISABLE_CRASH_DUMP` | 0 | crash dumps written to `ErrorLogs\*.dmp` | a dump is a memory image: may hold credentials; retention and access to that folder; ship to the platform, not kept on the host |
| 2 | `IP:PRIMARY` | 0 | the server's role in an A/B pair | semantics of 0/1, what changes on failover |
| 3 | `License:Demonstration_Key` | set | a demo licence key in clear text | still used with VIA License Manager? remove in production |
| 4 | `License:SLA` | absent (template only) | ? | meaning |
| 5 | `VIRTUAL:BIND_ADDRESS` … `FOURTH_BIND_ADDRESS` | 0.0.0.0 ×4 | the interfaces the server listens on | bind to the control VLAN only, never all interfaces; what each of the four binds serves |
| 6 | `VIRTUAL:CURRENT_VIRTUAL_ADDRESS` ×3 | absent (template only) | the virtual IP of the A/B pair | how the VIP moves (gratuitous ARP?), what the network must allow |
| 7 | `SQL:USE_DEFAULT` | 1 | use the local default instance | 0 when the database is on its own host |
| 8 | `SQL:LOGIN_CREDENTIALS` | 0 (template: 2) | 0 = Windows authentication, 2 = SQL login | the exact list of modes; Windows authentication with a managed account avoids any password in the file |
| 9 | `SQL:LOGIN_USERNAME`, `SQL:LOGIN_PASSWORD` | empty | the SQL login when mode 2 | **password in clear text in the file**: is there an encrypted form; otherwise file ACL to the session account only and the value from Vault |
| 10 | `SQL:PRIMARY_CONNECTION`, `SQL:SECONDARY_CONNECTION` | `localhost\CEREBRUMSQL` ×2 | the instance and its mirror partner | TLS on the SQL connection (encrypt=yes, trust server certificate no) |
| 11 | `SQL:REDUNDANT` | 0 | mirroring on/off | requirements (SQL Standard, Witness) |
| 12 | `SQL:DATABASE`, `SQL:SECONDARY_DATABASE` | `Cortex` ×2 | the database names | — |
| 13 | `SQL:USE_MULTI_SUBNET_CLUSTERING` | 0 | multi-subnet failover | supported SQL high-availability (mirroring vs Always On) |
| 14 | `COR_LINK:BIND_ADDRESS` | 0.0.0.0 | the Cortex link bind (server-to-server) | restrict; which port and peers |
| 15 | `CORTEX:RUNNING` | 1 | runtime state written by the application | not a setting: the role ignores drift on it |

### 12.2 The database table `dbo._Config` — everything else (33 categories, rows of user / category / property / value)

| # | Category | What it holds | Control / question |
|---|---|---|---|
| 16 | `LOGGING` | `USE_LOGGING_SERVER`, `SERVER_1_ADDRESS/PORT`, `SERVER_2_*`, `TRANSPORT_TYPE`, `USE_SSL`, `MESSAGES_TO_LOG` | the syslog target: TLS (`USE_SSL`) to the platform listener, both servers set; the format (non-standard columns) documented by the vendor |
| 17 | `SNMP` | agent port 1161, default version and communities, v3 auth/priv passwords, trap receiver port 162, poll time | communities and v3 passwords sit in the database in clear text: v3 with auth+priv only, no default community, the values from Vault |
| 18 | `TRAP` | trap servers 1 and 2, ports, communities, notify on startup | traps to the platform receiver; same community rule |
| 19 | `AUDIT` | per-area levels (application, protocol, SQL, UDP, services, web client, macros, scheduler, IP router, ACP) | the levels that give user attribution of every change; what each level writes |
| 20 | `ALARM`, `EVENT_LOG` | what is logged as events (clients, GPI/GPO, UMD, salvos, scheduler, traps) | the set the operations need in Loki |
| 21 | `SECURITY` | ? (not in the broadcaster's export) | what it governs: client authentication, password policy, session timeout, LDAP |
| 22 | `WEB_CLIENT` | web client settings | TLS and authentication of the web client (ports 9080/9081) |
| 23 | `EMAIL` | SMTP for notifications | the platform mailbox, TLS, credentials from Vault |
| 24 | `SQL` | log archiving, sizes | retention of the database's own log tables |
| 25 | `CRASH_DUMP` | thread ids to the audit trail | — |
| 26 | `SERVICES`, `CLIENTS`, `GUI`, `SOUND`, `SCHEDULER`, `TALLY_ROUTER`, `IP`, `IP_RTR`, `VIRTUAL`, `COR_LINK`, `CORTEX`, `ACP`, `GENERIC_DEV`, `PROJECT`, `RC`, `MacroFile`, `STARTUP_UPDATE_AUTO_ACK`, `TABLE_RESTORE`, `IMPORT_ENABLE` | operational settings | exported as a whole from the lab once a read-only login exists; the role keeps them as inventory data |
| 27 | `Admin / File Paths` | backup path and other paths | the backup folder on a monitored volume |

### 12.3 Data files beside the application (XML, UTF-16)

| # | File | Lab content | Control / question |
|---|---|---|---|
| 28 | `Global Data Files\USERS.XML` | groups and users of the application: one group `Admin`, one user `Admin` of type Cerebrum | where passwords are kept (not in this file); LDAP/Authentik for people; per-user accounts for attribution, never a shared `Admin` |
| 29 | `Global Data Files\CLIENTS.XML` | known clients: name, address, type, last seen, MAC | an inventory the register can read |
| 30 | `Global Data Files\INDEX.XML`, `MENUS.XML` | the index of these files, navigation | — |
| 31 | `LOCAL DATA FILES\LOCAL SETTINGS.XML` | the local UI state | — |
| 32 | `Data Files\User Configuration.xml` | roles `Admin`, `Operator`, `Engineer` with their monitors | the role model to map on Authentik groups |
| 33 | `Data Files\…`, `Variables\*.cvf`, `Configurations\*.ccf`, `Projects`, `Forms` | routers, assignment engine, variables, the site design, panels | versioned in git as the owner's design, backed up nightly by the application (`Auto Backup`) and by the guest backup |

### 12.4 Beside Cerebrum, on the same host

| # | Item | Lab | Control / question |
|---|---|---|---|
| 34 | Cerebrum SDN `etc\default_config.json` | `disableLicenseCheck`, bind option, API ports 5000/5002, HTTP API off | TLS floor 1.0 on the service (`--tls-min-v1.0`): raise; needed at all? |
| 35 | VIA License Manager `Config\ssl.conf`, `ldap.conf` | own certificate, key-store password in clear in the file, TLS 1.2 ciphers, LDAP off | LDAP against Authentik; the key-store password from Vault; a platform certificate |
| 36 | SQL Server instance | mixed authentication, dynamic port, Browser on, telemetry on, no backups | static port, Browser off, telemetry off, SQL login for the server only, backups to the NAS |
| 37 | The session | the server runs in the Administrator's RDP session | the service session account, auto-logon, Watchdog; RDP only through JumpServer |
| 38 | Firewall | "any port for Cerebrum.exe" | the explicit list of 12.1/§4 per program |

**Tracking and audit, the one question that matters for the vendor discussion:** every change an operator makes (crosspoint, salvo, macro, configuration) must land with the user's identity in a log the operator cannot edit. On the lab that is the audit-trail file on the host, writable by the same local Administrator who runs the application. The platform answer is syslog to Loki in real time plus per-user accounts in Cerebrum; the questions are rows 16, 19, 21 and 28.
