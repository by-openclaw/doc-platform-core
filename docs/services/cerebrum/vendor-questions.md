# Cerebrum — the installer's components and switches, to confirm with the vendor

What the platform knows about `Cerebrum_<version>_Installer.exe` (Inno Setup) comes from the broadcaster's playbooks and the lab; the vendor confirms or corrects each line. Checksums: the vendor publishes none; the platform computes the sha256 of each file it receives and pins that in the inventory.

## Components (`/COMPONENTS="a,b,c"`)

| # | Component | What we believe it installs | To confirm |
|---|---|---|---|
| 1 | `cerebrum` | the server application `Cerebrum.exe`, the web client files, the generic device library, the NMOS registry function, the SNMP agent | which Windows services, if any; whether the web client and the NMOS registry are inside it |
| 2 | `cerebrum_watchdog` | `Cerebrum Watchdog.exe`, the external watchdog started beside the server | its exact process name and whether it must run in the same session as the server |
| 3 | `cerebrum_designer` | the Designer (forms, panels, macros) | a seat licence? |
| 4 | `cerebrum_client` | the rich desktop client | — |
| 5 | `cerebrum_client_updater` | the client self-update service (file sync server→client) | its service name and port |
| 6 | `cerebrum_witness` | the Witness service of a redundant pair | its service name and port; needed only with a pair? |
| 7 | `cerebrum_database_mirror_creator` | the tool that sets SQL mirroring up for a pair | run once, or a service? |
| 8 | `cerebrum_service_manager` | `CerebrumService.exe`, registered as the Windows service `CerebrumServiceManager` with `MGMT_PORT=40009` (the Device Service) | the service's own ports beyond the management port |
| 9 | `cerebrum_federation` | `CerebrumFederationService.exe`, service `CerebrumFederationService` (ports 40003, 40030) | — |

## Switches

| Switch | What we believe | To confirm |
|---|---|---|
| `/InstallSQL=Install\|Skip` | installs the bundled SQL Server Express (instance `CEREBRUMSQL`, mixed mode, the documented `sa` password) with the ODBC driver and `sqlcmd` | can the instance name and the `sa` password be given on the command line; can the installer run with `/InstallSQL=Install` and **no** component (a database host alone) |
| `/InstallSentinel=Install\|Skip` | installs the Sentinel driver for a hardware licence key | — |
| `/VERYSILENT /SUPPRESSMSGBOXES /NORESTART /SP- /LOG=…` | Inno Setup's silent switches | the exit code that means "reboot required" |
| WebView2 runtime | the broadcaster installs it **before** Cerebrum as a dependency | bundled in 2.9.x, or still a prerequisite |

## Other questions already open (README §14.6, §15)

- The unattended import of a `.ccf` (command line, API, or UI only).
- A vendor import path for the settings table (`_Config`) besides SQL.
- TLS on the NMOS registry (BCP-003-01) and the certificate the Northbound API uses with `USE_SSL=1`.
- Whether `External Logging` licensing gates the per-service syslog.
- The IP-Core service logging the database credentials at INFO level (a defect report).
- Incremental installs: re-running the installer with more components keeps the existing ones intact?
