# How this compares to STIG and CIS baselines

WinLogs is **not** a STIG or CIS implementation and does not try to be. Those baselines mix two things: *what to log* and *how to harden the system*. This project aims to log **at least everything they require, and more**, and it deliberately ignores the hardening parts that get in the way of a disposable lab.

Sources checked: the DISA Windows 11 STIG, the RHEL 9 STIG, the macOS 14/15 STIGs, and CIS Windows 11 Benchmark v5 (via public mirrors of the benchmark text, not DISA's own files, so confirm against the release you care about). Only the items below were confirmed; this is a comparison of the logging requirements we could read, not a certification.

## What we log beyond the baselines
- **Windows:** Success and Failure on every audit subcategory that can fail, per-object audit rules on persistence and tamper-prone registry keys and folders, PowerShell Script Block, Module and Transcription logging for Windows PowerShell and PowerShell 7, Sysmon, and a long list of operational channels (WinRM, RDP, LSA, DeviceGuard, SMB, NTLM, AppLocker, PrintService, ...).
- **Linux:** the Neo23x0 best-practice ruleset (every `execve`, file, network, kernel-module, BPF and container rules), plus privileged-execution keys, with synchronous audit writes and a persistent, unthrottled journal.
- **macOS:** Endpoint Security events through `eslogger` (a wider set than the legacy audit flags), including task-port access, ptrace, permission changes and authorization decisions.

## Where we deliberately differ

| Topic | Baselines | WinLogs | Why |
|---|---|---|---|
| Security log size (Windows) | CIS: at least 196,608 KB. Current STIG: one week of records (its example is 5,120,000 KB); older STIG: 1,024,000 KB | 1 GiB (1,048,576 KB) by default; `-LogSizeMB` raises it | Meets CIS and the older STIG figure. Raise `-LogSizeMB` for a week of a busy host. |
| Audit rules immutable (Linux STIG) | `-e 2`, rules locked until reboot | `-e 1` | Rules must stay editable and revertable in a lab. |
| Disk full action (Linux STIG) | halt / single-user | distro default | Halting a machine mid-detonation destroys the evidence. |
| Central forwarding (CIS, STIG) | forward events to a collector | not done | Labs are usually air-gapped; forwarding is a separate stage. |
| macOS audit system | legacy OpenBSM flags in `audit_control` (`aa`, `ad`, `lo`, ...) and `expire-after:7d` | `eslogger` (Endpoint Security) | Apple deprecated the audit system (macOS 11) and disables it from macOS 14; `eslogger` is the supported replacement. Retention is by size, not days. |
| Log-size scanners (Windows) | read the Event Log *policy* value | we set both the channel and the policy value | Policy overrides the channel setting, so a policy refresh cannot shrink our logs. |

A STIG or CIS scanner will still report findings against this setup for the hardening items above. That is expected.
