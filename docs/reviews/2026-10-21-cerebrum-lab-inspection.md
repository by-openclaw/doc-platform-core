# Cerebrum lab inspection — the staging guest on the second hypervisor (2026-10-21)

**Status:** read-only inspection through the hypervisor's guest agent; nothing was changed. Input for the service ADR, the `Cerebrum.ini` template, the firewall data and the logging design. Companion of the repository review (same PR).

## 1. The guest

| Item | Found |
|---|---|
| Guest | `vm-cerebrum-stg-01`, 8 vCPU, 64 GiB, 250 GiB disk, one NIC on the hypervisor's management bridge, QEMU agent up |
| OS | Windows Server 2022 Standard **Evaluation** (build 20348), installed 2026-04-29, workgroup, local Administrator only, RDP on, WinRM on, no SSH |
| Time | `time.windows.com` (not the platform's firewall), Brussels time zone |
| Session | Cerebrum Server runs as the **Administrator's disconnected RDP session** (session 2), started by hand after the last reboot; no auto-logon, no scheduled task, no watchdog running |
| Lab hygiene | CCleaner (with scheduled tasks), Apple Software Update, Azure Arc tray, Wireshark + Npcap, Edge auto-update, SQL CEIP telemetry on: none of it belongs on a production server |

## 2. What is installed (versions as of 2026-10-21)

| Component | Version | How it runs |
|---|---|---|
| Cerebrum (server, Watchdog, Designer, Client, Client Updater, Witness, Database Mirror Creator, Service Manager, Federation Service, SDN installer, Sentinel init) | 2.9.3 (installed 2026-10-21; 2.9.2 before, 2.8.11 on 2026-06-23) | `Cerebrum.exe` as a desktop process in an interactive session |
| Cerebrum SDN (`ipcore`) | 2.8.3 | Windows service through `nssm`, `--tls-min-v1.0`, an embedded Prometheus (TSDB under the program folder) |
| EVS VIA License Manager | service, own SQLite database, TLS 1.2 with its own certificate, LDAP authentication available but off | the licensing model in use: **not** the Sentinel/echoid dongle flow of the broadcaster's repo (Sentinel driver 7.6.1 is installed but no Sentinel service runs; `Licenses\lservrc` is empty; `Cerebrum.ini` carries a demonstration key) |
| SQL Server 2019 **Express** (15.0.2190.7), named instance `CEREBRUMSQL` | mixed authentication, TCP on a **dynamic** port, SQL Browser on, SQL Agent disabled (Express), database `Cortex` 200 MB, **no backups in the backup folder** | Cerebrum connects as `localhost\CEREBRUMSQL`, database `Cortex`, Windows authentication (`SQL:USE_DEFAULT=1`, `LOGIN_CREDENTIALS=0`) |
| Dependencies | Edge WebView2 155, ODBC Driver 17, OLE DB Driver 18, SQL Native Client 2012, VC++ 2022 runtimes, Bonjour (mDNS) | |

## 3. Listening ports (the firewall data)

| Port | Protocol | Owner | Meaning |
|---|---|---|---|
| 9081 | TCP | Cerebrum.exe | web client / RCP (rule "Cerebrum RCP HTTP" is on 9080 via http.sys) |
| 40001, 40002 | TCP + UDP | Cerebrum.exe | server protocol (clients, panels) |
| 40009 | TCP | Cerebrum.exe | Service Manager link |
| 162 | UDP | Cerebrum.exe | SNMP trap receiver |
| 1161 | UDP | Cerebrum.exe | SNMP agent of Cerebrum |
| 2000, 2072 | UDP | Cerebrum.exe | device protocols (to identify with the vendor) |
| 8080 | TCP, http.sys | `x-nmos/registration`, `x-nmos/query` reservations | NMOS registry hosted by Cerebrum; Bonjour on 5353 for discovery |
| 9080 | TCP, http.sys | rule "Cerebrum RCP HTTP" | |
| 80 | TCP, http.sys | `Temporary_Listen_Addresses` only | Windows default, not Cerebrum |
| 5000, 5002 | TCP | ipcore (SDN) | IS-06 API and gRPC (HTTP API on 5001 disabled) |
| 8558 | TCP | VIA License Manager | licence API/UI |
| 1434 UDP, dynamic TCP | sqlbrowser, sqlservr | SQL named instance |
| 3389, 5985, 135/139/445 | | Windows | RDP, WinRM, SMB |

Firewall today: all profiles on, default inbound block, with program-wide **allow-everything** rules for `Cerebrum.exe`, `ipcore.exe` and the SDN's `prometheus.exe`, plus explicit 8080, 8558, 9080. The platform's data model keeps the explicit ports and drops the "any" rules once the vendor confirms the list.

## 4. Where things live (for the templates and the backup design)

| What | Path under `C:\Program Files\EVS Broadcast Equipment\Cerebrum\` |
|---|---|
| Main configuration | `Cerebrum.ini` (SQL connection, bind addresses up to four, Cortex link, licence key, `USE_MULTI_SUBNET_CLUSTERING`) — keys beyond the broadcaster's template |
| System configuration (the "design" of a site) | `Configurations\*.ccf`, auto-backed-up nightly at 02:30 to `Configurations\Auto Backup\AutoBackup_<date>.ccf` and `LastShutdown.ccf` (about 630 KB each, 31 kept) |
| Projects / forms (panels, UI packages) | `Projects\Common\…`, `Forms\{Application,Device,Hardware Panels,User}` (1,700 device forms shipped) |
| Device library | `GenericDevices\<family>\…` (xml, yaml), `SNMPDevices`, `SNMP Templates`, `AppMIBs`, `UserMIBs` |
| Data files | `Data Files\…` (routers, assignment engine, macros, user configuration), `Global Data Files\{CLIENTS,USERS,MENUS,INDEX}.XML`, `LOCAL DATA FILES\LOCAL SETTINGS.XML`, `Variables\*.cvf` |
| Scripts | `Python Scripts\*.py` (device drivers written in Python, run by Cerebrum itself) |
| Logs | `ErrorLogs\*.dmp` (crash dumps), `ErrorLogs\AuditTrail\AuditTrail_<date>.log` (tab-separated: no., time, date, address, source, event, severity, sub-item, server; 10 MB rotation; the live file is locked by the process) |
| Backups | `Backup\` (examples only), the SQL backup folder is empty |
| Updates | `Application Updates\` (client files pushed by the Client Updater) |
| Web client | `WebClient\`, `Web Client Files\` |
| SDN | `..\Cerebrum SDN\etc\{default_config.json, default_logging_config.json, prometheus.yml}`, logs to stdout only |
| Licence manager | `..\Via License Manager\{data\vialicense.db, Config\{ldap.conf, ssl.conf, certificates\}}` |

No Windows event-log provider exists for Cerebrum (only the licence manager and SQL Server). Logging for the platform therefore means: tail the audit-trail files with Alloy (tab-separated, one line per event, severity column), watch the crash-dump folder, read the SDN service's stdout through `nssm`'s log redirection, and SQL Server's errorlog. Cerebrum itself emits SNMP traps rather than syslog.

## 5. What the lab tells the design

1. **Service account and session.** Cerebrum Server needs an interactive session. The lab runs it as Administrator in a left-open RDP session: not reproducible. The platform role must create the session account, the auto-logon and the start-at-logon tasks (server + watchdog), and the CIS exception covers that host only.
2. **SQL.** Express is in use and works; set a **static TCP port**, disable the Browser, keep Windows authentication for the local server, SQL login only when server and database are on different hosts (then the login lives in Vault and lands in the `.ini`); turn off CEIP; add SQL-native backups of `Cortex` to the NAS and keep the `.ccf` auto-backups in the guest backup.
3. **Licensing.** VIA License Manager replaces the dongle flow of the broadcaster's repo: a service with its own database and certificate. How licences are issued to it (file, online activation, per host or floating) is a vendor question.
4. **NMOS and discovery.** Cerebrum hosts the NMOS registry on 8080 and relies on Bonjour (mDNS) for discovery; on a routed platform network that needs unicast DNS-SD or a per-subnet choice.
5. **SDN.** A separate product (2.8.3) with its own embedded Prometheus and TLS 1.0 floor: only if the IP network orchestration is wanted; then its ports and TLS floor are decisions.
6. **Time.** The guest syncs to a public NTP server; the platform's firewall is the time source for every host.
7. **Nothing on the lab is a secret worth keeping:** evaluation OS, demo licence key, local Administrator; the lab is a reference, not a template.

## 6. Missing — to look up by hand or to ask the vendor

| # | Question | Who |
|---|---|---|
| 1 | The official port list per component (server, Service Manager, clients, web client, NMOS, SDN, licence manager), including UDP 2000/2072 and the SNMP ports seen | vendor documentation / support |
| 2 | Licensing with VIA License Manager: issuance (file or online), per host or floating, transfer between hosts, offline activation, behaviour when the licence manager is down | vendor |
| 3 | Supported way to run Cerebrum Server unattended: auto-logon account, Watchdog, RDP constraints, whether a Windows service mode exists in 2.9 | vendor |
| 4 | SQL Server: supported editions and versions for 2.9.3, Express limits for the intended size, static-port guidance, backup and restore procedure of `Cortex`, maintenance | vendor + the owner's sizing |
| 5 | Client types to be used: desktop client, web client (9081/9080), Designer seats, hardware panels; the device list of the lab (what the server will control) | owner |
| 6 | Logging: audit-trail format reference, severity levels, whether Cerebrum can emit syslog or only SNMP traps; rotation settings | vendor |
| 7 | SDN: is it needed; its TLS floor; its Prometheus scrape list | owner + vendor |
| 8 | NMOS discovery on routed networks: unicast DNS-SD support, registry HA | vendor |
| 9 | High availability: A/B servers, Witness, Database Mirror Creator requirements (SQL Standard), failover behaviour | vendor, only if HA is wanted |
| 10 | Upgrade path and installer switches (the repo's `/COMPONENTS`, `/InstallSQL`, `/InstallSentinel` flags; 2.9.2 → 2.9.3 was done in place on the lab) | vendor documentation |
| 11 | Windows Server licence for the platform hosts (the lab runs an evaluation) | owner |
| 12 | Installer files and checksums for 2.9.3, WebView2, ODBC/OLE DB, SQL Express 2019, VIA License Manager, SDN | owner (vendor download area) |
