# Cerebrum — the playbooks, one per scope

One playbook per concern, each the full lifecycle of its scope (install, settings as data, secrets from Vault, firewall, logging, liveness, backup line, verification), run in the order below. Every task is an Ansible module call; the broadcaster's 46 inline PowerShell scripts are not carried over. Hosts and tiers are inventory data; nothing below names an address. Secrets live in Vault under `<tier>/cerebrum/<host>/<key>` and reach the host as module arguments, never on a command line. "Setting" numbers refer to the list in [README §15](README.md#15-where-the-three-kinds-of-cerebrum-state-live-and-the-findings-of-2026-10-22).

| Seq | Playbook | Hosts (group) | Role | Scope in one line |
|---|---|---|---|---|
| 1 | `cerebrum-db.yml` | `cerebrum_db` (vm-winsql-01) | `cerebrum_db` | the SQL instance Cerebrum will create its database in |
| 2 | `cerebrum-license.yml` | `cerebrum_license` (vm-winapp-02) | `cerebrum_license` | VIA License Manager, certificate, licence |
| 3 | `cerebrum-server.yml` | `cerebrum_server` (vm-winapp-01) | `cerebrum_server` | the server application, its ini, data files, auto-start, Watchdog |
| 4 | `cerebrum-db-config.yml` | `cerebrum_db` | — (tasks) | the settings rows of `_Config` as data, export and converge |
| 5 | `cerebrum-device-service.yml` | `cerebrum_device_service` (vm-winapp-02) | `cerebrum_device_service` | the Device Service (Service Manager) |
| 6 | `cerebrum-nmos.yml` | `cerebrum_nmos` (vm-winapp-02) | `cerebrum_nmos` | the NMOS registry and discovery |
| 7 | `cerebrum-web.yml` | `cerebrum_server` | `cerebrum_web` | the web client, RCP and hardware panels |
| 8 | `cerebrum-client.yml` | `cerebrum_clients` (Win11 Pro seats) | `cerebrum_client` | the desktop client and Designer seats |
| 9 | `cerebrum-sdn.yml` | `cerebrum_server` | `cerebrum_sdn` | IP-Core, only when IP routing is used |
| 10 | `cerebrum-watch.yml` | `monitoring` (Linux) | `cerebrum_watch` | the Northbound subscription to syslog and Prometheus |
| 11 | `cerebrum-ops.yml` | all of the above | — (tasks, `-e op=`) | start, stop, version report, export, import, failover, support capture |

Prerequisites on every Windows host, applied by `windows-guests.yml` before any of these: `windows_guest`, `windows_hardening`, `windows_firewall`, `windows_logging`. The platform firewall rules (OPNsense catalog) are a separate PR per zone, before seq 1.

---

## 1. `cerebrum-db.yml` — the SQL instance

| # | Step | How | Value from | Check |
|---|---|---|---|---|
| 1.1 | SQL Server 2019 Express (or Standard for a redundant pair) from the object store, checksum pinned | `win_package` with the vendor's unattended options (named instance `CEREBRUMSQL`, mixed mode) | `cerebrum_db_version`, `cerebrum_db_instance` | `win_service_info` MSSQL$CEREBRUMSQL running |
| 1.2 | `sa` password set at install, never the vendor's default (README §15 finding 5) | installer argument from Vault, `no_log` | `<tier>/cerebrum/<host>/sa` | login test with `mssql_script` (`SELECT 1`) |
| 1.3 | TCP enabled on a **static** port; dynamic ports off; SQL Browser disabled | `win_regedit` on the instance's `SuperSocketNetLib\Tcp` keys, `win_service` | `cerebrum_db_port` | `win_powershell`-free: `win_wait_for` on the port |
| 1.4 | Logins: application (dbcreator until the database exists, then db_owner of `Cerebrum`), IP-Core (read/write on the tables the help lists), audit read-only (`db_datareader`) | `mssql_script`, passwords from Vault as module arguments | `<tier>/cerebrum/<host>/app-login`, `/ipcore-login`, `/audit-ro` | `mssql_script` `SELECT name FROM sys.sql_logins` |
| 1.5 | Services: Agent off, CEIP/telemetry service off, VSS writer on | `win_service` | standard | service states |
| 1.6 | Maintenance: recovery model, max sizes (settings 22), nightly native backup to the platform share, verify | SQL Agent is absent on Express: `win_scheduled_task` running `sqlcmd` with a maintenance script **from the role's files**, or the platform backup of the guest plus the application's own archiving | `cerebrum_db_backup_*` | a backup file younger than 24 h |
| 1.7 | Firewall: the SQL port from the server host only; nothing else | `win_firewall_rule` with `remoteip` | inventory | rule listing |
| 1.8 | Redundant pair: mirroring in high-safety mode with a witness, or Always On, per README §8 | later, its own tasks file | HA decision | `Redundancy_Status.SQL_*` over the API |
| 1.9 | Liveness: TCP probe of the port; logs: SQL errorlog to Loki | `windows_logging` channel | — | Loki query |

## 2. `cerebrum-license.yml` — VIA License Manager

| # | Step | How | Value from | Check |
|---|---|---|---|---|
| 2.1 | VIA License Manager 3.0.4 from the object store | `win_package` | `cerebrum_license_version` | service running on 8558 |
| 2.2 | TLS 1.2 only, certificate from the platform CA instead of the self-signed one (`ssl.conf`) | `win_template` of `Config\ssl.conf`, certificate via the certificate role | Step-CA | `openssl s_client` from the controller (`uri` with `validate_certs: true`) |
| 2.3 | `ldap.conf` stays off (identity through the platform, not the licence manager) | `win_template` | standard | file content |
| 2.4 | The licence: file or key activation as the vendor delivers it; Sentinel driver only for a hardware key | `win_copy` from Vault for a file, `win_package` for the driver | `<tier>/cerebrum/license` | License tab objects over the API: `License.License_Type` not Demo, `Supported_Options.*` |
| 2.5 | Firewall: 8558 from the server hosts and the admin jump only | `win_firewall_rule` | inventory | rule listing |
| 2.6 | Backup line: the SQLite store of the licence manager in the guest backup | documented | — | — |

## 3. `cerebrum-server.yml` — the server application

| # | Step | How | Value from | Check |
|---|---|---|---|---|
| 3.1 | `Cerebrum_<version>_Installer.exe` from the object store, checksum pinned, product id known, components declared (`/COMPONENTS=…`), no `/InstallSQL`, no `/InstallSentinel` | `win_package` | `cerebrum_version`, `cerebrum_components` | registry product present, version |
| 3.2 | `Cerebrum.ini`: the 15 keys (settings 8 to 11): remote database, dedicated login, bind addresses, crash dump, no demo key | `win_template`, password from Vault inside the template variables, `no_log` | inventory + `<tier>/cerebrum/<host>/app-login` | file diff |
| 3.3 | Data files (setting 31): `Global Data Files\*.XML`, `Data Files\*.xml`, `LOCAL DATA FILES\LOCAL_SETTINGS.XML`, `Variables\*.cvf` | `win_template` (UTF-16 where the vendor's files are) | role templates + inventory | file diff |
| 3.4 | Accounts in `Global Data Files\USERS.XML`: Admin (password from Vault), the API monitoring user (read-only), the ops user (write) | `win_template` | `<tier>/cerebrum/<host>/admin`, `/nb-api-monitor`, `/nb-api-ops` | login over the API with each |
| 3.5 | `GenericDevices`: the declared family set only, every other file removed (every file loads, vendor checklist row 7) | `win_copy` from the role's files + `win_find`/`win_file` for the rest | `cerebrum_device_families` | folder listing equals the declaration |
| 3.6 | Projects and forms: the site's `.cjf` and `Forms` from the object store | `win_get_url` + `win_unzip`-free copy (`.cjf` is one file) | `cerebrum_project` | file present, hash |
| 3.7 | `GlobalMacros`: the vendor's `WatchdogHealthCheck.cmf` and the site's macros | `win_copy` | role files | — |
| 3.8 | Auto-start: Cerebrum and the Watchdog application at logon of the service session, RDP session settings (vendor checklist row 14) | `win_scheduled_task`, `win_regedit` | standard | tasks listed, process present after a reboot |
| 3.9 | Services the installer registers (Service Manager link) with restart-on-failure | `win_service` | standard | service states |
| 3.10 | Firewall per port: 40001/40002 tcp+udp, 40009 tcp, 2071/2072 udp, 162/1161 udp; program-wide "any" rules removed | `win_firewall_rule` | README §4 | rule listing |
| 3.11 | Logs: `ErrorLogs\AuditTrail\*.log` tailed, `ErrorLogs\*.dmp` alerted; `Export Today's Logs` path (setting 22) | `windows_logging` extra jobs | — | Loki query |
| 3.12 | First start: the application creates the database and the default `_Config` rows; then seq 4 | `win_scheduled_task` run once, `win_wait_for` on 40001 | — | `_Config` row count over the audit login |
| 3.13 | Backup line: nightly `Auto Backup` (setting 23) to a path outside the Cerebrum folder, copied to the object store by seq 11 | settings + ops | — | a `.ccf` younger than 24 h in the store |

## 4. `cerebrum-db-config.yml` — the settings as data

| # | Step | How | Value from | Check |
|---|---|---|---|---|
| 4.1 | Export the live `_Config` to the controller (user, category, property, value), secrets masked | `mssql_script` with the audit login, `copy` to a dated file | `<tier>/cerebrum/<host>/audit-ro` | file |
| 4.2 | Declared rows: settings 12 to 28 from `inventories/<tier>/group_vars/cerebrum.yml` (`cerebrum_config_rows`), the broadcaster's standard as the role default | role defaults + inventory | README §15 | — |
| 4.3 | Converge: diff declared against live, `UPDATE`/`INSERT` only the differences, with the application login | `mssql_script`, `changed_when` on the diff | — | second run = 0 changes |
| 4.4 | Rows never written: setting 29 (`IP/INSTANCE_*`, `SECURITY/KEY`, `GUI/LAST_PASSWORD`) | explicit exclusion list | — | — |
| 4.5 | Cerebrum restarted after a change (settings are read at start) | `cerebrum-ops.yml -e op=restart` | — | API login after restart |

## 5. `cerebrum-device-service.yml` — the Device Service

| # | Step | How | Value from | Check |
|---|---|---|---|---|
| 5.1 | Installer with the Device Service component only | `win_package` | `cerebrum_version` | — |
| 5.2 | Service registration `CerebrumServiceManager` with `MGMT_PORT` = setting 15, restart-on-failure | `win_service` (`path` with arguments), `win_service` failure actions | inventory | service running |
| 5.3 | Firewall: the management port from the server, the device protocols (XS via Linx 31150–31155, video mixer 62010) from the device zones | `win_firewall_rule` | README §4 | rule listing |
| 5.4 | Registration on the server: the Service Manager appears under `02: Services › SERVERS[*]`; its services `STATE=Running` | API read after the first start | — | `cerebrum-watch` alarm template |
| 5.5 | Failover with a second Device Service, when there is one | `FAILOVER_ENABLED` on the device rows (seq 11 op) | HA decision | `DATA_CONNECTION_HEALTH` |

## 6. `cerebrum-nmos.yml` — the NMOS registry

| # | Step | How | Value from | Check |
|---|---|---|---|---|
| 6.1 | Bonjour from the object store (`BonjourPSSetup.exe`) for mDNS | `win_package` | pack | service running |
| 6.2 | The registry (vendor's NMOS component) on 8080, the URL reservation it needs (`netsh http add urlacl` as the broadcaster did, through `win_command` with argv) | `win_package`, `win_command` | `cerebrum_nmos_port` | `uri` on `x-nmos/registration` and `x-nmos/query` |
| 6.3 | DNS-SD records on the platform resolver for the zones where mDNS does not reach (README §7) | the DNS chain, separate PR | inventory | `dig _nmos-register._tcp` |
| 6.4 | Firewall: 8080 tcp and 5353 udp from the media zones | `win_firewall_rule` | README §4 | rule listing |
| 6.5 | The device row in Cerebrum (`DEVICE_CONFIG`, type Network Media Server) through the `.ccf` of seq 11 | — | site export | `01: IP Router` objects over the API |

## 7. `cerebrum-web.yml` — the web client, RCP and panels

| # | Step | How | Value from | Check |
|---|---|---|---|---|
| 7.1 | Settings 16 and 17: `WEB_CLIENT` (ENABLED, PORT, WEB_SERVER_PORT, USE_SSL=1, idle timeout, MAX_CONNECTIONS), `RCP` (ENABLE_RCP, HTTP_PORT, USE_SSL), `PROXY` | rows of seq 4 (`cerebrum_config_rows` web subset) | inventory | — |
| 7.2 | The certificate for the web client (`SECURITY/CERTIFICATE`, `CERT_HASH`) from the platform CA, in the Windows store | certificate role, `win_certificate_store`, the thumbprint written as the setting | Step-CA | browser check through Traefik |
| 7.3 | Web client files and `Projects\Common\WebClient` from the project | part of seq 3.6 | — | `uri` on the web client URL |
| 7.4 | Hardware panels: the `.cpf` set of the site, `HW_PANEL` settings | `win_copy`, rows | site files | panel visible in the Clients table |
| 7.5 | Exposure: through the platform edge with SSO where the web client allows it, else private | Traefik route, separate PR | README §11 | — |

## 8. `cerebrum-client.yml` — the desktop seats

| # | Step | How | Value from | Check |
|---|---|---|---|---|
| 8.1 | Installer with the client (and Designer where declared) components | `win_package` | `cerebrum_client_components` per host | product present |
| 8.2 | Client data files: `User_Configuration.xml`, `LOCAL_SETTINGS.XML`, the server address, `GUI` settings for clients (setting 25) | `win_template` | inventory | file diff |
| 8.3 | Firewall: outbound to the server ports only; inbound nothing | `win_firewall_rule` | README §4 | rule listing |
| 8.4 | Version equal to the server (vendor checklist rows 18, 20) | `cerebrum-ops.yml -e op=version` | — | report |
| 8.5 | The tooling the owner asked for on the seats (browser, Wireshark with the connector's dissectors, VS Code with the language extensions) | the Windows profile of infra/0011 | PR #75 | — |

## 9. `cerebrum-sdn.yml` — IP-Core, when IP routing is used

| # | Step | How | Value from | Check |
|---|---|---|---|---|
| 9.1 | Cerebrum SDN package from the object store; service `cerebrumipcore` through nssm with stdout/stderr files **and** rotation (`AppRotateFiles`, `AppRotateBytes`) | `win_package`, `win_regedit` on the nssm parameters | `cerebrum_sdn_version` | service running, log size bounded |
| 9.2 | `etc\default_config.json`, `default_logging_config.json`, `prometheus.yml` | `win_template` | inventory | file diff |
| 9.3 | Its own SQL login (never `sa`), TLS minimum raised from the vendor's `--tls-min-v1.0` | login of seq 1.4, service parameter | Vault | log shows the login |
| 9.4 | The IP-Core log is not shipped as-is: the lines carrying the database configuration are dropped or masked by Alloy (README §15 finding 1) | `windows_logging` pipeline stage | — | Loki query shows no credential line |
| 9.5 | Firewall: 5000 (IS-06), 5001, 5002 (gRPC) from the controller zones | `win_firewall_rule` | README §4 | rule listing |

## 10. `cerebrum-watch.yml` — the subscription

| # | Step | How | Value from | Check |
|---|---|---|---|---|
| 10.1 | The connector binary (go-acp release, checksum) on the monitoring host | `get_url` + `file` | `cerebrum_watch_version` | `dhs version` |
| 10.2 | One systemd unit per server and sub-device: `watch --device 0.0.0.0 --sub-device N --object <list>` with the monitoring user from Vault as environment of the unit (not argv), `--syslog-addr` to Alloy, `--metrics-addr` for Prometheus, `--alarm` with the vendor severities | `template` of the unit, `systemd` | README §14.3 and §14.4 | unit active, Loki lines, Prometheus target up |
| 10.3 | Per controlled device a unit with its own object list | `cerebrum_watch_devices` | inventory | — |
| 10.4 | Alert rules from README §14.3 | Prometheus rules through the monitoring role | — | rule listing |

## 11. `cerebrum-ops.yml` — operations, `-e op=`

| # | `op` | What it does | Check |
|---|---|---|---|
| 11.1 | `version` | version of server, clients, Device Service, licence manager into the register | report |
| 11.2 | `start` / `stop` / `restart` | the application and Watchdog on the server, the Device Service, in the vendor's power sequence (help topic *server power sequencing*) | API login, `ALL_RESPONSIVE` |
| 11.3 | `export` | a dated `.ccf` export plus the settings export of seq 4.1 to the object store with access control (the export holds the user table) | file in the store, decodable (README §15.1) |
| 11.4 | `import` | the site's `.ccf` into a server, `IMPORT_ENABLE` set for the tables allowed (setting 27); unattended path: **vendor question**, until then the documented manual step | device rows over the API |
| 11.5 | `failover` | the Device Service failover and the redundant pair switch through the API's writable objects (`FAILOVER`, `ACTIVATE_DEVICE`) with the ops user | `Redundancy_Status`, `DATA_CONNECTION_HEALTH` |
| 11.6 | `support-capture` | the vendor's *Export Today's Logs* plus the audit trail and the IP-Core log to the object store for a ticket (vendor checklist row 23) | archive present |
| 11.7 | `services` | start, stop or disable a service row under `02: Services` through the API | `SERVICES[*].STATE` |

---

## What is decided here, what is not

- Decided by the owner on 2026-10-21 and 2026-10-22: one playbook per scope, everything in Ansible, Vault for every secret, no provisioning on the WS2025 guests until the template is tuned, IP-Core and the gauges parked, the database provisioned by the platform.
- Vendor questions that gate a step: the unattended `.ccf` import (11.4), the Device Service's exact package and registration (5.1, 5.2), whether `External Logging` licensing gates the per-service syslog, whether the settings table has a vendor import path besides SQL (4.3).
- Open on the platform side: the read-only SQL login on the lab (parked), the `sa` rotation on the lab (owner), infra/0011 (#75) for the seats' tooling.
