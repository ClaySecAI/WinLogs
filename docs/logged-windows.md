# Windows: what gets logged

See also: [Linux](logged-linux.md) · [macOS](logged-macos.md) · [overview](what-gets-logged.md)

| Area | Setting | Where to look |
|---|---|---|
| Audit policy | `windows/audit.csv` via `auditpol /restore` (logon, account/group mgmt, process creation/termination, object access, policy change, sensitive privilege use, ...) | Security log |
| Command lines | `ProcessCreationIncludeCmdLine_Enabled=1` | Security 4688 |
| Legacy override | `SCENoApplyLegacyAuditPolicy=1` so advanced audit policy wins | n/a |
| PowerShell | Script Block (incl. invocation start/stop), Module (`*`), Transcription to `C:\ProgramData\WinLogs\transcripts`; set for Windows PowerShell and PowerShell 7 | Microsoft-Windows-PowerShell/Operational 4103/4104, transcript files |
| Sysmon | sysmon-modular balanced config (process, network, DNS, image load, registry, WMI, pipes, ...) | Microsoft-Windows-Sysmon/Operational |
| Channels enabled / enlarged | Task Scheduler, DNS-Client, DriverFrameworks-UserMode (usually off); Defender, WMI-Activity, BITS-Client, Firewall, TerminalServices, CodeIntegrity, NTLM, WinRM, RDP client and RdpCoreTS, LSA, DeviceGuard, Netlogon, SMB Server, Shell-Core, AppX deployment, Winlogon, Security-Mitigations, PrintService, SmbClient Security, AppLocker (4 logs), Diagnosis-Scripted, PowerShellCore, System, Application | each channel |
| Sizes | Security, Sysmon, PowerShell: `-LogSizeMB` (default 1024). Others: up to 256 MB. Logs are never shrunk. Security, System and Application also get the Event Log *policy* size value, which overrides the per-channel setting, so a policy refresh cannot shrink them. | n/a |

## Client vs Server

`Install-WinLogs.ps1` picks an audit profile automatically (`-AuditProfile Auto|Workstation|Server`):

| | Windows 10 / 11 | Windows Server (incl. domain controllers) |
|---|---|---|
| Base policy (`audit.csv`) | yes | yes |
| Kerberos Authentication Service and Service Ticket Operations (4768/4769/4771...), Directory Service Access, Directory Service Replication (`audit-server.csv`) | no (these only generate events on a DC) | yes |
| Extra channels | n/a | Directory Service, DNS Server, DNS Server Audit, DFS Replication (only if the role is installed) |

## Audit policy
Everything we audit that can fail is audited for **Success and Failure**: failed attempts (denied access, lockouts, rejected IPsec, system faults) are often the first sign of something wrong.
- Added over the original 2016 baseline: PNP Activity (USB / device insertion, 6416), Group Membership (4627), Other Object Access Events (scheduled task creation 4698/4702), SAM (4661), Other Policy Change Events, MPSSVC Rule-Level Policy Change (firewall rule changes 4946-4948), DPAPI Activity (4692/4693), Removable Storage, and the IPsec Main / Quick / Extended Mode categories.
- Turned from Success-only to Success and Failure: Account Lockout, IPsec Driver, Other System Events, Detailed File Share, File System, Registry.
- Still off on purpose because they are very noisy: Handle Manipulation, Non Sensitive Privilege Use, Other Privilege Use, Filtering Platform Packet Drop, Detailed Directory Service Replication.
- Filtering Platform Connection stays Success only (failures are every blocked connection).

## Per-object auditing (SACLs)
The File System and Registry audit categories only log access to objects that carry an audit rule. `windows/sacls.psd1` lists what gets one (success audit for Everyone) and is meant to be edited:
- Registry: Run / RunOnce (native and WOW6432Node), Policies\Explorer\Run, Active Setup, Winlogon, Windows (AppInit_DLLs), Services, Session Manager, Image File Execution Options, SilentProcessExit, Lsa, SecurityProviders (WDigest), and the Defender policy key. Events: 4657 (value change), 4656/4663. Rules audit failed attempts as well as successes.
- Files: the all-users StartUp folder, `System32\Tasks`, `System32\drivers`, and the hosts file. Events: 4663 (and 4656).
- Per-user hives (HKCU Run keys) are not covered, since each user's hive is only loaded while that user is signed in.
- Tamper-protected keys (for example Defender) may refuse the rule; that is reported as a warning and the run continues.
- `-SkipObjectAuditing` leaves these out; `-Revert` removes exactly the rules it added.

Notes
- Filtering Platform Connection (5156/5158) is very noisy and overlaps Sysmon event 3. Remove that row from `audit.csv` if logs roll too quickly.
- Transcripts are plain files that malware running as admin can delete. Fine for a lab, not evidence-grade.
- Everything is recorded in `C:\ProgramData\WinLogs\state.json`; `-Revert` restores it.
