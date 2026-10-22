# Review — the two Cerebrum Ansible repositories (received 2026-10-21)

**Status:** review for the owner's decision (step 5 of the agreed sequence); nothing from these repositories has been executed.
**Input:** two archives, extracted into their own quarantine directories on the controller; imported as read-only archives on the platform GitLab (`platform/incoming`).
**Purpose:** say what they install and how, measure them against the platform rules, and propose the scopes we build ourselves.

## 1. What the archives contain

| Archive | Content | Size |
|---|---|---|
| A — the playbooks | 67 top-level playbooks + 13 `Dev/` + 10 `Test/`: per site (Central, Playout, FCR, HVA, NOC, SI, Global, Migration) × environment (prod, preclip, dev, test) × action (install, start, stop); directory provisioning; database configuration export/import; a version report; a NetBox install | 127 YAML files, 11 inventories |
| B — the roles | `windows-cerebrum` (installer, `Cerebrum.ini`, firewall rules, scheduled tasks, Service Manager service, SQL checks, licence locking-code tool, log cleanup) and `windows-nmos` (HTTP URL reservations and the NMOS service) | 9 YAML, 2 templates |

**Missing from both archives:** the generic Windows roles (`sv/*`, 31 roles) that every playbook composes: firewall, SNMP, nxlog, WinRM, Defender, updates, 7-Zip, Notepad++, Firefox, Wireshark, Sysinternals, VLC, Python 3.8, SQL ODBC driver, theme, BGInfo, groups, directory objects. They sit in a third repository of another team. We do not need them: the platform has its own equivalent for each (§4).

## 2. How the application is deployed there

- **Connection and identity:** WinRM over Kerberos with a domain administrator typed at every run (`vars_prompt`), machines joined to an Active Directory; access groups and service sessions are AD groups and users; one playbook creates the directory objects.
- **Installers:** downloaded from an internal HTTPS distribution server with TLS verification disabled (`ssl_validation: false`): the vendor installer (versions 2.5.29 → 2.9.3, a table of file versions for idempotence), the Edge WebView2 runtime, the licence locking-code tool, SQL Server 2019 Express, the Sentinel driver, SQL Server Management Studio.
- **The server is a desktop application, not a service:** Cerebrum Server and its Watchdog start in an auto-logged-on user session through a scheduled task at logon; start/stop playbooks drive them with inline PowerShell. Only the Service Manager (device service) is a Windows service.
- **Configuration:** `Cerebrum.ini` from a template (SQL login, database, bind addresses, an optional demo licence key); the application's own configuration table (`dbo._Config`) exported to YAML and written back with the MS SQL modules.
- **High availability as designed by the vendor:** two servers per site (A/B) with SQL mirroring tools (Witness, Database Mirror Creator) — this needs SQL Server Standard, not Express.
- **Secrets:** HashiCorp Vault lookups in the test playbooks only; prompts elsewhere; no encrypted files; the SQL password reaches the `.ini` in clear on the host (the application needs it so).
- **Scripts:** 46 inline PowerShell/shell tasks, one `.ps1` deployed for log cleanup, an NMOS start script run from an SMB share.
- **Network:** per-program firewall rules — server 80/443, 8080–8085 (FCR API), 8090–8095 (SAMIX API), 40003 and 40030 (federation), 40007 (mediaconf), 40051 (exchanger); Service Manager 40009, 31150–31155, 62010 and a management port; NMOS 8080; plus "any port for the program" rules on the server and the Service Manager. Defender exclusions for the executables.

## 3. Against the platform rules

| Rule | Finding | Verdict |
|---|---|---|
| Ansible only, no scripts (`infra/0010`) | 46 inline scripts, a deployed `.ps1`, a script from a share | rewrite with modules: scheduled tasks as data, `win_service`, `win_find`/`win_file`, the MS SQL modules |
| Secrets in Vault, never on a command line or prompt (`security/0001`) | prompts for the domain admin; Vault only in tests | Vault documents under `cerebrum/…` and under the identities; the `.ini` is the one place the SQL login must land (file 0600-equivalent ACL, class `system`) |
| Identity: local accounts, SSH keys, Authentik; no directory (`identity/0004`) | AD groups, Kerberos, domain session user | drop the directory playbook; the session account becomes a local account whose password lives in Vault; people reach the hosts through JumpServer |
| Pinned, verified installers through the proxy (`infra/0011`) | vendor installers from an internal server, TLS off | the EVS installers go to the platform object store with checksums; TLS stays on |
| One firewall rule per application and port (`infra/0011`, Windows walk) | explicit ports **and** "any port" program rules | keep the explicit ports as data; no "any" rule unless the vendor documents why |
| CIS L1 baseline (Windows walk) | auto-logon session, no screen lock, Defender exclusions | a documented exception for the **server host only** (auto-logon service session), decided in the service ADR |
| Hardening and agents already on the hosts | nxlog, SNMP, WinRM, BGInfo, theme | not needed: Alloy → Loki, windows_exporter, SSH |
| Applications | 7-Zip, Notepad++, Firefox, Wireshark, Sysinternals, VLC, Python 3.8, ODBC driver, SSMS | an `infra/0011` profile `cerebrum` |
| NetBox playbook | installs a NetBox | drop, the platform has one |

## 4. Scopes we build (one role and one playbook each, new code, the repos as reference only)

| # | Scope | Host (proposal) | What it carries |
|---|---|---|---|
| 1 | `cerebrum-db` | vm-winsql-01 | SQL Server 2019 Express (or Standard, §5), instance, database, application login (Vault), ODBC driver, SSMS, port 1433 to the server only, SQL-native backup to the NAS beside the guest backup |
| 2 | `cerebrum-server` | vm-winapp-01 | WebView2, VLC, the vendor installer with the chosen components, `Cerebrum.ini` from Vault, firewall rules per port, the service session account + scheduled tasks, Defender exclusions, log cleanup as a scheduled task with a plain command, the licence locking code |
| 3 | `cerebrum-device-service` | vm-winapp-02 | the Service Manager as a Windows service, its ports |
| 4 | `cerebrum-client` | workstations (Win11 Pro, later) | client, client updater, Designer where allowed |
| 5 | `cerebrum-nmos` | vm-winapp-02 | URL reservations, the NMOS registration/query service on 8080 |
| 6 | `cerebrum-db-config` | playbook | export/import of the application's `_Config` table as inventory data |
| 7 | `cerebrum-ops` | playbooks | start/stop of the session applications, the version report feeding the register |

## 5. Decisions for the owner

| # | Decision | Options | My recommendation |
|---|---|---|---|
| 1 | SQL edition | Express (10 GB per database, no mirroring) / Standard (licence, mirroring, the vendor's A/B design) | Express now on the lab host; Standard when a second server exists |
| 2 | The server's session account | local account, auto-logon, CIS exception on that host / run only what can be a service | local account from Vault, exception documented in the service ADR |
| 3 | Which site profile first | Central, Playout, FCR, HVA, NOC, SI | the one you will actually drive; Central is the smallest |
| 4 | Target version | 2.9.3 is the newest in their table | 2.9.3, pinned |
| 5 | Licence material | Sentinel key, locking codes per host | you provide, stored in Vault under `cerebrum/license` |
| 6 | NMOS | yes / no | only if a device needs it |
| 7 | Python 3.8 on servers | their tooling needs it / not needed | not installed unless a component needs it |
| 8 | Inputs you provide | EVS installers (exact files), licence, the components per host, the first site's parameters | before step 7 |

Nothing starts before these are answered; then the service ADR (`services/00xx-cerebrum`) and the `infra/0011` profile, then the roles, proven on vm-winapp-02 first.
