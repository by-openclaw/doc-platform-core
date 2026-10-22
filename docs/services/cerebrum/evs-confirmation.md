# Cerebrum provisioning — values per concern, for EVS confirmation

One section per concern, one small table each. Columns: the setting, the value the platform will set, and what EVS confirms or corrects (empty = our choice, no answer needed). Placeholders in curly braces are site values from the inventory. Sources: *standard* = identical on the broadcaster's three systems, *lab* = staging server 2.9.3, *vendor* = checklist v2.01 or help file, *CIS* = SQL Server 2019 benchmark.

## 1. Installer

| # | Setting | Value we set | EVS confirms |
|---|---|---|---|
| 1.1 | Package | `Cerebrum_2.9.3_Installer.exe`, silent: `/VERYSILENT /SUPPRESSMSGBOXES /NORESTART /SP- /LOG=…` | the switches; the exit code meaning "reboot required" |
| 1.2 | Components, server | `cerebrum, cerebrum_watchdog` | component names; Windows services each one installs |
| 1.3 | Components, client seat | `cerebrum_client` (+ `cerebrum_designer` on a Designer seat) | is `cerebrum_client_updater` needed; its service and port |
| 1.4 | `/InstallSQL` | `Skip` always (SQL Server from Microsoft, patched by Microsoft) | supported SQL versions and editions for 2.9.x |
| 1.5 | `/InstallSentinel` | `Skip` (software licence) or `Install` (hardware key) | how a software licence is applied unattended |
| 1.6 | Prerequisite | Microsoft Edge WebView2 runtime installed first | still required in 2.9.x, or bundled |
| 1.7 | Upgrade | same silent install over the existing version; ini and data kept | supported in-place path 2.8.x → 2.9.3; steps before/after |

## 2. SQL Server (one instance, shared by the Cerebrum servers)

| # | Setting | Value we set | EVS confirms |
|---|---|---|---|
| 2.1 | Edition, instance | Microsoft SQL Server 2019 Express (Standard for a redundant pair); named instance `CEREBRUMSQL`; mixed mode | Express acceptable in production for a standalone |
| 2.2 | Network | static TCP 1433; dynamic ports off; Browser off; encryption forced with the platform certificate | the ODBC driver works with `ForceEncryption=1` |
| 2.3 | Logins | `sa` disabled; one application login per Cerebrum server, owner of its own database only; one read-only audit login | rights needed beyond the database (the help lists `sys.*` views for a remote instance) |
| 2.4 | Databases | one per Cerebrum server (`Cortex_A`, `Cortex_B`), created by Cerebrum at first start; a redundant pair shares one | two independent servers may share one instance with separate databases |
| 2.5 | Hardening (CIS §2) | off: ad hoc queries, CLR, xp_cmdshell, OLE automation, remote access, Database Mail, cross-db chaining; login auditing both outcomes | any option Cerebrum needs on |
| 2.6 | Backup | native full backup nightly 02:00, 7 rotating files, verified | |
| 2.7 | Redundancy (later) | mirroring high-safety with witness, or Always On | which one for 2.9.x; Standard edition required |

## 3. Cerebrum.ini

| # | Key | Value we set | EVS confirms |
|---|---|---|---|
| 3.1 | `SQL:USE_DEFAULT` | `0` | |
| 3.2 | `SQL:LOGIN_CREDENTIALS` | `2` | meaning of the values |
| 3.3 | `SQL:LOGIN_USERNAME` / `SQL:LOGIN_PASSWORD` | the server's own login; password from Vault | the password is in clear in the file: an encrypted form exists? |
| 3.4 | `SQL:PRIMARY_CONNECTION` | `{db-host}\CEREBRUMSQL,1433` | the `host\instance,port` syntax |
| 3.5 | `SQL:DATABASE` / `SQL:SECONDARY_DATABASE` | the server's database name | |
| 3.6 | `SQL:REDUNDANT` / `SQL:SECONDARY_CONNECTION` / `SQL:USE_MULTI_SUBNET_CLUSTERING` | `0` / same as primary / `0` on a standalone | |
| 3.7 | `VIRTUAL:BIND_ADDRESS` … `FOURTH_BIND_ADDRESS`, `COR_LINK:BIND_ADDRESS` | the server's own address (`0.0.0.0` on the lab) | when a bind address must be set (multi-homed) |
| 3.8 | `IP:PRIMARY`, `CORTEX:DISABLE_CRASH_DUMP`, `CORTEX:RUNNING` | `0`, `0`, `1` | |
| 3.9 | `License:Demonstration_Key` | empty on a licensed server | |

## 4. Settings table (`dbo._Config`) — core

| # | Category / property | Value we set | EVS confirms |
|---|---|---|---|
| 4.1 | `CORTEX/DISABLE_CTL_NAV_TALLY_ON_SERVER` | `1` (vendor row 3) | |
| 4.2 | `CORTEX/USE_STRONG_PASSWORD` | `1` | |
| 4.3 | `CORTEX/IS_REDUNDANT`, `INDEPENDENT_CONFIGS`, `ALLOW_VIRTUALISE` | `0`, `0`, `0` | |
| 4.4 | `CRASH_DUMP/ENABLE_WATCHDOG`, `PROTOCOL_PROTECTION`, `WATCHDOG_ABORT_AFTER_FAILOVER` | `1`, `1`, `1` (vendor row 6) | |
| 4.5 | `CRASH_DUMP/EXTENDED`, `WRITE_THREAD_IDS_TO_AUDIT` | `0`, `0` | |
| 4.6 | `SERVICES/SERVICE_MANAGEMENT_PORT` | `40009` | same port as the Northbound API? |
| 4.7 | `COR_LINK/MULTICAST_ADDRESS`, `MULTICAST_PORT`, `FILETRANS_PORT` | `239.1.1.1`, `40001`, `40006` | |
| 4.8 | `COR_LINK/HEALTH_FAILOVER_GRACE_PERIOD`, `MAX_RESENDS`, `SEND_MULTICAST_PACKET` | `60000`, `4`, `0` | `0` acceptable with panels on another subnet |
| 4.9 | `COR_LINK/SERVER_NAME` | `{server-name}` | |
| 4.10 | `FEDERATION/USE_FEDERATION` | `0` | |

## 5. Settings — logging, SNMP, mail, audit

| # | Category / property | Value we set | EVS confirms |
|---|---|---|---|
| 5.1 | `LOGGING/USE_LOGGING_SERVER` | `1` | **External Logging** licence required for this? |
| 5.2 | `LOGGING/SERVER_1_ADDRESS`, `SERVER_1_PORT` | `{alloy-host}`, `1514` | |
| 5.3 | `LOGGING/TRANSPORT_TYPE`, `USE_SSL`, `MESSAGES_TO_LOG` | `1`, `1` if supported, `2` | meaning of `TRANSPORT_TYPE` (UDP/TCP) and `MESSAGES_TO_LOG` (0–2); syslog format (RFC 3164 or 5424) and field order |
| 5.4 | `SNMP/AGENT_PORT`, `DEFAULT_VERSION`, `POLL_TIME`, `TRAP_RX_PORT` | `1161`, `v2c`, `3`, `162` | SNMPv3 supported? |
| 5.5 | `SNMP/RO_COMMUNITY_STRING`, `RW_COMMUNITY_STRING` | from Vault; write community empty | agent can be read-only? |
| 5.6 | `TRAP/TRAP_SERVER`, `TRAP_PORT`, `TRAP_COMMUNITY`, `TRAP_VERSION` | `{monitoring-host}`, `162`, from Vault, `2c` | |
| 5.7 | `TRAP/NOTIFY`, `ON_STARTUP` | `1`, `1` | |
| 5.8 | `EMAIL/SMTP_SERVER`, `SMTP_PORT`, `SECURITY` | `{mail-host}`, `587`, STARTTLS | `SECURITY` values; authenticated submission on 587 supported |
| 5.9 | `EMAIL/SMTP_USERID`, `SMTP_PASSWORD` | the service mailbox; password from Vault | |
| 5.10 | `EMAIL/FROM_ADDRESS`, `SENDER_NAME`, `ERROR_TO_ADDRESS` | `cerebrum@{domain}`, `{server-name}`, `{ops-mailbox}` | |
| 5.11 | `EMAIL/NOTIFY`, `NOTIFIY_CRITICAL_ERROR`, `PERIODIC_REPORT` | `1`, `1`, `0` | |
| 5.12 | `AUDIT` levels | `ACP 3`, `APP 1`, `FILE 1`, `PROT 1`, `SERVICES 3`, `SQL 1`, `UDP 1`, `WEB_CLIENT 1`, others `0` (standard) | recommended production levels; cost of level 3 |
| 5.13 | `AUDIT/RX_BUFFER_LOG_SIZE`, `RX_QUEUE_LOG_LENGTH` | `1024`, `1000` | |
| 5.14 | `ALARM` switches | `LOG_CLIENT_EVENTS 1`, `LOG_GD_CONTROL 1`, `LOG_GPI_CHANGES 1`, `LOG_GPO_CHANGES 1`, `LOG_RX_TRAPS 1`, `LOG_SCHEDULER_EVENTS 1`, `LOG_UMD_CHANGES 1`, `NOTIFICATION_MODE 1`, `REACTIVATE_ON_ERROR 1`, `PAL 1`, others `0` (standard) | |

## 6. Settings — storage, archiving, backup

| # | Category / property | Value we set | EVS confirms |
|---|---|---|---|
| 6.1 | `SQL/ARCHIVE_DIRECTORY` | `D:\Cerebrum\Archive` (outside the program folder, vendor row 11) | |
| 6.2 | `SQL/LOG_ARCHIVING`, `TRAIL_ARCHIVING` | `1`, `1` | |
| 6.3 | `SQL/MAX_DB_APP_SIZE`, `MAX_DB_LOG_SIZE`, `MAX_DB_TRAIL_SIZE` | `512`, `512`, `128` | units (MB); sizes for 48 h of history on a busy site |
| 6.4 | `SQL/MAX_LOG_ARCHIVE_SIZE`, `MAX_TRAIL_ARCHIVE_SIZE` | `4096`, `4096` | |
| 6.5 | `SQL/FORCE_TRUNCATE`, `AUTO_FAILOVER` | `1`, `0` | |
| 6.6 | `TALLY_ROUTER/AUTO_BACKUP`, `BACKUP_DIRECTORY`, `BACKUP_PERIOD` | `1`, `D:\Cerebrum\Backup`, `24` | |
| 6.7 | `TALLY_ROUTER/BACKUP_LIMIT`, `BACKUP_SIZE` | `14`, `30` (standard has limit 1) | meaning of both |
| 6.8 | `Admin/File Paths`: Backup Files, `EXPORT_TODAYS_LOGS`, `AUDITLOG_FILE`, `SQLXML_FILE` | all under `D:\Cerebrum\…` | |

## 7. Settings — web client, RCP, TLS

| # | Category / property | Value we set | EVS confirms |
|---|---|---|---|
| 7.1 | `WEB_CLIENT/ENABLED`, `USE_SSL` | `1`, `1` | elevated run required with TLS (help): any alternative |
| 7.2 | `WEB_CLIENT/WEB_SERVER_PORT`, `PORT` | `443`, `40002` | |
| 7.3 | `WEB_CLIENT/ENABLE_IDLE_TIMEOUT`, `IDLE_TIMEOUT`, `MAX_CONNECTIONS`, `MAC_ADDRESS_IDENTIFICATION`, `DISTRIBUTED_HOSTING` | `1`, `1800000`, `40`, `1`, `0` | |
| 7.4 | `RCP/ENABLE_RCP`, `HTTP_PORT`, `USE_SSL` | `1`, `9080`, `1` | |
| 7.5 | `SECURITY/CERTIFICATE`, `CERT_HASH` | path and thumbprint of a certificate from the platform CA, CN/SAN = server FQDN (virtual address on a pair) | store or file; renewal without downtime |
| 7.6 | `SECURITY/KEY` | left as Cerebrum writes it | |
| 7.7 | `PROXY/USE_PROXY`, `PROXY_FQDN`, `WEB_CLIENT_SERVICE_IP_CSV` | `0`, empty, empty (the platform edge terminates TLS in front) | reverse proxy in front of the web client supported |

## 8. Settings — GUI, import, defaults, never written

| # | Category / property | Value we set | EVS confirms |
|---|---|---|---|
| 8.1 | `GUI/AUTO_LOGON`, `DEMO_MODE_LOGON`, `WINDOWS_CREDENTIALS`, `CONFIRM_SHUTDOWN`, `SYSTEM_TRAY_MESSAGE` | `1`, `0`, `1`, `1`, `1`; display switches standard | which account `AUTO_LOGON` uses |
| 8.2 | `IMPORT_ENABLE` (what a `.ccf` import may overwrite) | running system: `DEVICES 1`, `DEVICE_DETAILS 1`, others `0`; fresh server: all `1` | recommended set for a restore on a fresh server |
| 8.3 | `IP_RTR`, `GENERIC_DEV`, `SCHEDULER`, `HW_PANEL/STARTUP_DELAY` | standard; standard; standard; `60` | |
| 8.4 | `IP/INSTANCE_1`, `IP/INSTANCE_2`, `GUI/LAST_PASSWORD`, `SECURITY/KEY` | never written by the platform | generated per server, never copied between servers |
| 8.5 | the table itself | exported, diffed and written by SQL (user, category, property, value); Cerebrum restarted after | a vendor import/export path besides SQL |

## 9. Accounts and data files

| # | Item | Value we set | EVS confirms |
|---|---|---|---|
| 9.1 | `Global Data Files\USERS.XML` | `Admin` (password from Vault); `monitor` read-only for the Northbound API; `ops` write for the Northbound API | how to create users unattended; a read-only group for the API |
| 9.2 | `CLIENTS.XML`, `INDEX.XML`, `MENUS.XML`, `User_Configuration.xml`, `LOCAL_SETTINGS.XML`, `Variables\*.cvf` | templated from the lab's files (UTF-16) | |
| 9.3 | `GenericDevices\*.cgd` | only the declared families plus the server's own 8 files; the rest removed (vendor row 7) | files that must never be removed |

## 10. Configuration (`.ccf`) and Northbound API

| # | Item | Value we set | EVS confirms |
|---|---|---|---|
| 10.1 | `.ccf` export | nightly Auto Backup plus a dated export to the object store (it holds the user table) | unattended import on a fresh server: command line, API or UI only |
| 10.2 | `DEVICE_CONFIG` control interface | Northbound API (XML), WebSocket server `40009`, `USE_SSL=1` (lab: `0`) | certificate used with `USE_SSL=1`; port configurable |
| 10.3 | Northbound licences | 10 sessions on the lab; platform uses 3 (watch) + 1 (ops) | |

## 11. NMOS registry

| # | Item | Value we set | EVS confirms |
|---|---|---|---|
| 11.1 | Bonjour for Windows | installed (mDNS, DNS-SD) | |
| 11.2 | URL reservations | `netsh http add urlacl http://*:8080/x-nmos/registration/` and `/x-nmos/query/`, user Everyone | |
| 11.3 | Firewall | 8080 tcp and 5353 udp inbound from the media zones | |
| 11.4 | DNS-SD | `_nmos-register._tcp` and `_nmos-registration._tcp` records on the platform resolver, one registry | |
| 11.5 | TLS | none (help: HTTP) | BCP-003-01 (TLS) supported; Query API for third parties |

## 12. Host firewall (per program, Cerebrum.exe)

| # | Direction | Value we set | EVS confirms |
|---|---|---|---|
| 12.1 | Inbound | 40001, 40002 tcp+udp (clients, panels); 40009 tcp (Northbound); 2071, 2072 udp (ACP); 162 udp (traps); 1161 udp (SNMP agent); 443, 9080 tcp (web, RCP); 8080 tcp, 5353 udp (NMOS) | ports seen in the broadcaster's rules and not here: 2000, 40003, 40030, 40006, 40007, 40051, 8080–8085, 8090–8095 — which ones a standalone server needs |
| 12.2 | Outbound | SQL to the database host; ACP and SNMP to the device ranges; syslog, traps, mail to the platform hosts; nothing else | any other outbound need (licence server, updates) |

## 13. Logging and service account

| # | Item | Value we set | EVS confirms |
|---|---|---|---|
| 13.1 | Audit trail, error logs | `ErrorLogs\AuditTrail\*.log` shipped to Loki; `.dmp` alerted; 30 days kept | |
| 13.2 | IP-Core (`cerebrumipcore`) | not installed (IP routing not used) | defect: the service logs the SQL login and password at INFO on every ini re-read |
| 13.3 | Windows session hosting Cerebrum and Watchdog | local account `cerebrum`; logon task starts Watchdog then server; Administrators only while TLS requires it | recommended account and privilege; supported auto-logon method |

## 14. Licence, Device Service, monitoring, backup, time

| # | Item | Value we set | EVS confirms |
|---|---|---|---|
| 14.1 | Cerebrum licence | Sentinel key or software licence; options seen on the lab: clients 10, web clients 10, Northbound 10, SNMP, Redundant server, External Logging | what **External Logging** gates; unattended application of a software licence |
| 14.2 | Device Service (later) | `CerebrumService.exe … MGMT_PORT=40009`, restart on failure; 31150–31155 and 62010 only with those devices | component name; its own ports; registration on the server |
| 14.3 | Monitoring | Northbound subscription on `00: System` and `02: Services`, to syslog and Prometheus; thresholds CPU < 15 %, queues 0, run rate < 2000 (vendor row 12) | thresholds; `Clients` and `CorLinkStats` as C-Link health |
| 14.4 | Backup | nightly guest image (PBS) plus 2.6, 6.6, 10.1 | |
| 14.5 | Time | platform NTP on every host, Brussels time zone (vendor row 14) | |
