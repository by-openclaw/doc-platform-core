# Cerebrum — the playbooks, one per scope

Seven playbooks are needed now; two are kept for later; one is not needed. Each is the full lifecycle of its scope (install, settings as data, secrets from Vault, firewall, logging, liveness, backup line, verification), every task an Ansible module call, hosts and tiers inventory data, secrets as module arguments only. "Setting n" refers to the numbered list in [README §15](README.md#15-where-the-three-kinds-of-cerebrum-state-live-and-the-findings-of-2026-10-22). Decisions by the owner on 2026-10-22: IP-Core/SDN is not used; the VIA License Manager licenses Neuron, not Cerebrum; the NMOS registry is Bonjour, firewall and DNS as the vendor's IS-04/05 help describes.

| Seq | Playbook | Hosts (group) | Needed | Scope in one line |
|---|---|---|---|---|
| 1 | `cerebrum-db.yml` | `cerebrum_db` | now | the SQL instance Cerebrum creates its database in |
| 2 | `cerebrum-server.yml` | `cerebrum_server` | now | the server application, its licence, ini, data files, auto-start, Watchdog |
| 3 | `cerebrum-db-config.yml` | `cerebrum_db` | now | the settings rows of `_Config` as data |
| 4 | `cerebrum-nmos.yml` | `cerebrum_server` | now | the NMOS registry: Bonjour, URL reservations, firewall, DNS-SD |
| 5 | `cerebrum-web.yml` | `cerebrum_server` | now | the web client and RCP over TLS, hardware panels |
| 6 | `cerebrum-client.yml` | `cerebrum_clients` | now | the desktop and Designer seats |
| 7 | `cerebrum-ops.yml` | all | now | start, stop, version, export, import, support capture |
| 8 | `cerebrum-watch.yml` | `monitoring` | with 7 | the Northbound subscription to syslog and Prometheus |
| 9 | `cerebrum-device-service.yml` | `cerebrum_device_service` | later | the Device Service, when a second control point or a remote registry is wanted |
| — | IP-Core / SDN | — | **not used** | the lab's `cerebrumipcore` service is a vendor leftover; the roles do not install it |
| — | VIA License Manager | — | **not Cerebrum** | licenses Neuron; it belongs to the Neuron scope, not to these playbooks |

Before seq 1: the platform baseline on every Windows host (`windows-guests.yml`: `windows_guest`, `windows_hardening`, `windows_firewall`, `windows_logging`) and the OPNsense catalog rules for the Cerebrum ports (one PR per zone).

---

## 1. `cerebrum-db.yml` — the SQL instance

| # | Step | How | Value from | Check |
|---|---|---|---|---|
| 1.1 | SQL Server 2019 (Express, or Standard for a redundant pair) from the object store, checksum pinned, named instance, mixed mode | `win_package` with the unattended options | `cerebrum_db_version`, `cerebrum_db_instance` | the instance service running |
| 1.2 | `sa` password set at install from Vault, never the vendor's documented default (README §15 finding 5) | installer argument, `no_log` | `<tier>/cerebrum/<host>/sa` | `mssql_script` `SELECT 1` |
| 1.3 | TCP on a **static** port, dynamic ports off, SQL Browser off | `win_regedit` on the instance's network keys, `win_service` | `cerebrum_db_port` | `win_wait_for` on the port |
| 1.4 | TLS on the instance: a server certificate from the platform CA in the machine store, `ForceEncryption=1`; the lab has neither | certificate role + `win_regedit` (`SuperSocketNetLib`) | Step-CA | an encrypted login from the server host |
| 1.5 | Logins: the application (database creator until the database exists, then owner of it), the audit read-only (`db_datareader`) | `mssql_script`, passwords as module arguments | `<tier>/cerebrum/<host>/app-login`, `/audit-ro` | `sys.sql_logins` |
| 1.6 | Services: Agent off, telemetry off, VSS writer on | `win_service` | standard | service states |
| 1.7 | Sizes and archiving (setting 22), nightly native backup to the platform share | rows of seq 3 + `win_scheduled_task` running a maintenance file from the role | `cerebrum_db_backup_*` | a backup younger than 24 h |
| 1.8 | Firewall: the SQL port from the server host only | `win_firewall_rule` with `remoteip` | inventory | rule listing |
| 1.9 | Redundant pair later: mirroring in high-safety mode with a witness, or Always On (README §8) | its own tasks file | HA decision | `Redundancy_Status.SQL_*` over the API |

## 2. `cerebrum-server.yml` — the server application

| # | Step | How | Value from | Check |
|---|---|---|---|---|
| 2.1 | `Cerebrum_<version>_Installer.exe` from the object store, checksum pinned, components declared, no bundled SQL | `win_package` | `cerebrum_version`, `cerebrum_components` | product and version in the registry |
| 2.2 | The licence: Cerebrum is licensed through a Sentinel key (the License tab reads the key serial; the lab runs the demo key with none). Hardware key: the Sentinel driver from the installer. Software licence: the vendor's licence file applied as the help's *licence upgrade procedure* describes (*to confirm on the help topic*) | `win_package` for the driver, `win_copy` from Vault for a file | `<tier>/cerebrum/license` | `License.License_Type` not Demo, `Supported_Options.*` over the API |
| 2.3 | `Cerebrum.ini`: settings 8 to 11: remote database, dedicated login, bind addresses, crash dump, no demo key | `win_template`, `no_log` | inventory + `<tier>/cerebrum/<host>/app-login` | file diff |
| 2.4 | Data files (setting 31): `Global Data Files`, `Data Files`, `LOCAL DATA FILES`, `Variables` | `win_template` (UTF-16 where the vendor's files are) | role templates + inventory | file diff |
| 2.5 | Accounts (setting 30): Admin, the API monitoring user (read-only), the ops user (write) | `win_template` of `USERS.XML` | `<tier>/cerebrum/<host>/admin`, `/nb-api-monitor`, `/nb-api-ops` | an API login with each |
| 2.6 | `GenericDevices`: the declared family set only (every file loads, vendor checklist row 7) | `win_copy` + removal of the rest | `cerebrum_device_families` | folder equals the declaration |
| 2.7 | The site's project (`.cjf`) and `GlobalMacros` (`WatchdogHealthCheck.cmf`) | `win_copy` from the object store | `cerebrum_project` | hash |
| 2.8 | Auto-start of Cerebrum and the Watchdog at logon of the service session, **with highest privileges** (TLS binding needs it, see Certificates), RDP session settings | `win_scheduled_task`, `win_regedit` | standard | process present after a reboot |
| 2.9 | Firewall per port: 40001/40002 tcp+udp, 40009 tcp, 2071/2072 udp, 162/1161 udp; no program-wide "any" rule | `win_firewall_rule` | README §4 | rule listing |
| 2.10 | Logs: audit trail tailed, crash dumps alerted (setting 22 paths outside the Cerebrum folder) | `windows_logging` extra jobs | — | Loki query |
| 2.11 | First start: Cerebrum creates the database and the default `_Config` rows; then seq 3 | run-once task + `win_wait_for` on 40001 | — | row count over the audit login |
| 2.12 | Backup line: the nightly `Auto Backup` (setting 23) outside the Cerebrum folder, copied to the object store by seq 7 | settings + ops | — | a `.ccf` younger than 24 h in the store |

## 3. `cerebrum-db-config.yml` — the settings as data

| # | Step | How | Value from | Check |
|---|---|---|---|---|
| 3.1 | Export the live `_Config` (user, category, property, value) to a dated file, secrets masked | `mssql_script` with the audit login, `copy` | `<tier>/cerebrum/<host>/audit-ro` | file |
| 3.2 | Declared rows: settings 12 to 28 in `inventories/<tier>/group_vars/cerebrum.yml`; the broadcaster's standard as the role default | role defaults + inventory | README §15 | — |
| 3.3 | Converge: diff declared against live, write only the differences with the application login | `mssql_script`, `changed_when` on the diff | — | second run = 0 changes |
| 3.4 | Never written: setting 29 (`IP/INSTANCE_*`, `SECURITY/KEY`, `GUI/LAST_PASSWORD`) | exclusion list | — | — |
| 3.5 | Restart after a change (settings are read at start) | `cerebrum-ops.yml -e op=restart` | — | API login after restart |

## 4. `cerebrum-nmos.yml` — the registry Cerebrum hosts

From the vendor's *NMOS IS-04/05* help: Cerebrum hosts the IS-04 registration server when a **Network Media Server** device is added; it depends on Bonjour for Windows for mDNS and DNS-SD; registration mode works across subnets, peer-to-peer mode (mDNS only) does not; the registry answers the registration API on 8080 and does **not** serve the Query API to third parties; nodes heartbeat every 5 s and are dropped after 12 s.

| # | Step | How | Value from | Check |
|---|---|---|---|---|
| 4.1 | Bonjour for Windows from the object store (the pack's `BonjourPSSetup.exe`) | `win_package` | pack | Bonjour service running |
| 4.2 | URL reservations the help requires: `http://*:8080/x-nmos/registration/` and `http://*:8080/x-nmos/query/` for Everyone (the account name differs on non-English systems) | `win_command` with `netsh http add urlacl` as argv, `changed_when` on the listing | `cerebrum_nmos_port` | `netsh http show urlacl` |
| 4.3 | Firewall: 8080 tcp inbound from the media zones; 5353 udp for mDNS where nodes share the subnet | `win_firewall_rule` | README §4 | rule listing |
| 4.4 | DNS-SD for nodes on other subnets: `_nmos-register._tcp` and `_nmos-registration._tcp` PTR/SRV/TXT records on the platform resolver, one registry only | the DNS chain, separate PR | inventory | `dns-sd -B _nmos-register._tcp` from a node's subnet lists one entry |
| 4.5 | The Network Media Server device row in Cerebrum (through the `.ccf` of seq 7, or declared) | `.ccf` import | site export | `http://<server>:8080/x-nmos/registration` lists the versions; `01: IP Router › Senders/Receivers` over the API |
| 4.6 | Node labels and SubIDs kept in the IP Routing configuration, stored in the device datastore XML (replicated in SQL when the server hosts the registry) | documented, data in the `.ccf` | — | — |
| 4.7 | Later, with a Device Service: the registry hosted there with a service virtual IP shared by two Cerebrum systems | seq 9 | — | — |

## 5. `cerebrum-web.yml` — the web client, RCP and panels

| # | Step | How | Value from | Check |
|---|---|---|---|---|
| 5.1 | Settings 16 and 17: `WEB_CLIENT` (ENABLED, PORT, WEB_SERVER_PORT 443, USE_SSL=1, idle timeout, MAX_CONNECTIONS), `RCP` (ENABLE_RCP, HTTP_PORT, USE_SSL=1), `PROXY` | rows of seq 3 | inventory | — |
| 5.2 | The TLS certificate (see Certificates): from the platform CA, CN and SAN = the server FQDN (the virtual address for a redundant pair), in the machine store; `SECURITY/CERTIFICATE`, `CERT_HASH` written as settings | certificate role, `win_certificate_store`, rows | Step-CA | browser check through the edge |
| 5.3 | Web client files from the project (seq 2.7) | — | — | `uri` on the web client URL |
| 5.4 | Hardware panels: the site's `.cpf` set and `HW_PANEL` settings | `win_copy`, rows | site files | the panel in the Clients table |
| 5.5 | Exposure: the web client behind the platform edge (forwardAuth; the application keeps its own login, it has no OIDC) | Traefik route, separate PR | README §11 | — |

## 6. `cerebrum-client.yml` — the desktop seats

| # | Step | How | Value from | Check |
|---|---|---|---|---|
| 6.1 | Installer with the client (and Designer where declared) components | `win_package` | `cerebrum_client_components` per host | product present |
| 6.2 | Client data files: `User_Configuration.xml`, `LOCAL_SETTINGS.XML`, the server address, `GUI` settings for clients (setting 25) | `win_template` | inventory | file diff |
| 6.3 | Firewall: outbound to the server ports only | `win_firewall_rule` | README §4 | rule listing |
| 6.4 | Version equal to the server (vendor checklist rows 18, 20) | `cerebrum-ops.yml -e op=version` | — | report |
| 6.5 | The seats' tooling (browser, Wireshark with the connector's dissectors, VS Code with the language extensions) | the Windows profile of infra/0011 | PR #75 | — |

## 7. `cerebrum-ops.yml` — operations, `-e op=`

| # | `op` | What it does | Check |
|---|---|---|---|
| 7.1 | `version` | versions of server, clients and licence into the register | report |
| 7.2 | `start` / `stop` / `restart` | the application and Watchdog in the vendor's power sequence | API login, `ALL_RESPONSIVE` |
| 7.3 | `export` | a dated `.ccf` export plus the settings export (3.1) to the object store with access control (the export holds the user table) | file in the store, decodable (README §15.1) |
| 7.4 | `import` | the site's `.ccf` into a server, `IMPORT_ENABLE` set for the allowed tables (setting 27); unattended path: **vendor question**, documented manual step until then | device rows over the API |
| 7.5 | `support-capture` | *Export Today's Logs* plus the audit trail to the object store for a ticket (vendor checklist row 23) | archive present |
| 7.6 | `services` | start, stop or disable a service row under `02: Services` through the API, with the ops user | `SERVICES[*].STATE` |

## 8. `cerebrum-watch.yml` — the subscription

| # | Step | How | Value from | Check |
|---|---|---|---|---|
| 8.1 | The connector binary (go-acp release, checksum) on the monitoring host | `get_url` + `file` | `cerebrum_watch_version` | `dhs version` |
| 8.2 | One systemd unit per server and sub-device: `watch` with the monitoring user from Vault in the unit's environment (never argv), `--syslog-addr` to Alloy, `--metrics-addr` for Prometheus, `--alarm` with the vendor severities | `template` + `systemd` | README §14.3, §14.4 | unit active, Loki lines, target up |
| 8.3 | One unit per controlled device with its own object list | `cerebrum_watch_devices` | inventory | — |
| 8.4 | Alert rules from README §14.3 | monitoring role | — | rule listing |

## 9. `cerebrum-device-service.yml` — later

Installer with the Device Service component; service `CerebrumServiceManager` registered with the server's management port (setting 15) and restart-on-failure; firewall for the device protocols; registration visible under `02: Services › SERVERS[*]`; failover through the API's writable objects. Written when a second control point or a shared registry is wanted.

---

## Certificates — where TLS applies and what the platform does

The lab has **no certificate at all** in its machine store and no TLS anywhere: the web client, RCP, the Northbound API, SQL (`ForceEncryption=0`, no certificate) and the registry all run in clear. The platform CA (Step-CA) issues everything below; nothing self-signed, nothing bought. The seats and the server trust the platform CA through the Windows profile.

| # | Where | What the vendor says | What the platform does |
|---|---|---|---|
| C1 | Web client (HTTPS + WebSocket Secure) | an X.509 v3 certificate, CN and SAN = the server FQDN (the virtual address on a redundant pair), installed on the server; `WEB_CLIENT/USE_SSL`, `WEB_SERVER_PORT 443`; the application must then run **elevated** | a server certificate from Step-CA, in the machine store, its path and thumbprint as `SECURITY/CERTIFICATE` and `CERT_HASH`; the auto-start task with highest privileges; renewal by the certificate role, thumbprint rewritten |
| C2 | RCP (HTTP 9080) | `RCP/USE_SSL` | the same certificate; verified with `uri` |
| C3 | Northbound API (WebSocket 40009) | `USE_SSL` on the control-interface device row | the same certificate is the assumption, *to prove on the lab with the connector's `--tls`*; until proven the API stays on the management zone only |
| C4 | Devices Cerebrum connects to over HTTPS or mTLS (NMOS nodes with BCP-003-01, REST devices) | device rows carry `USE_SSL`, `USE_SSL_MTLS`, `IGNORE_CERT_ERRORS`, `MTLS_CLIENT_CERT_THUMBPRINT` | the platform CA in the server's Root store (`IGNORE_CERT_ERRORS=0`), a client certificate from Step-CA where a device demands mTLS, its thumbprint in the row |
| C5 | The NMOS registry (8080) | HTTP only in the IS-04/05 help | kept on the media control zone; BCP-003-01 (TLS) support: *vendor question* |
| C6 | SQL (the server to the instance) | nothing; the packs hold the vendor's `Unlock_sql_certificate_issue.ps1`, a hint of a known issue (*to read*) | a server-authentication certificate from Step-CA on the instance, `ForceEncryption=1`, the ODBC driver on the server encrypting without `TrustServerCertificate` |
| C7 | Hardware panels (C-Link 40001/40002) | proprietary; one certificate mention in the nVision panels topic (*to read*) | kept on the control zone; no TLS assumed |
| C8 | The web client behind the platform edge | — | Traefik terminates with the platform certificate and forwardAuth protects the path; the application's own login stays (no OIDC in the product; EVS's Foundation/Keycloak layer is off on the lab, *vendor question*) |
| C9 | VIA License Manager (8558, TLS 1.2, own `ssl.conf`) | — | Neuron scope, not here |

Three items wait for the lab's guest agent, which stopped answering during the last reads: the licence upgrade procedure (2.2), the SQL certificate script (C6), the hardware panels topic (C7).

---

## What is decided, what is not

- Decided (owner, 2026-10-21 and 2026-10-22): one playbook per scope, everything in Ansible, Vault for every secret, no provisioning on the WS2025 guests until the template is tuned, the database provisioned by the platform, IP-Core not used, VIA License Manager out of scope, the Device Service later.
- Vendor questions gating a step: the unattended `.ccf` import (7.4), TLS on the registry (C5), the Northbound certificate (C3), the Foundation identity layer (C8), whether `External Logging` licensing gates the per-service syslog.
- Platform items: the read-only SQL login on the lab (parked), the `sa` rotation on the lab (owner), PR #75 for the seats' tooling.
