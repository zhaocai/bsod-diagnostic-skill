---
name: bsod-diagnostic-skill
description: >-
  Diagnose Windows blue screen (BSOD / 蓝屏 / 系统突然重启 / 电脑自动重启) crashes
  from the System event log and minidumps using the standard Microsoft workflow.
  Use when the user reports a blue screen, sudden reboot, or asks why their PC
  crashed. Covers Event 1001 BugCheck / Event 41 Kernel-Power queries, minidump
  discovery, WinDbg (cdb) + Microsoft symbol server analysis with !analyze -v,
  IRP stack interpretation for 0x9F, and actionable recommendations.
---

# Windows BSOD Diagnostic

Standard, reproducible workflow for diagnosing Windows blue-screen crashes
from Event Viewer + minidumps with WinDbg/cdb. This mirrors Microsoft's official
"Bug Check" debugging process (analyze minidump with `!analyze -v` + symbol
server), with the practical Windows-environment plumbing solved (Store-installed
WinDbg ACL restrictions, quoting issues, encoding).

## Step 0 - Read this first

- The user is usually anxious and non-technical. Confirm the crash date/time
  FIRST, then explain in plain language. Do not jump to "hardware is broken".
- Almost all BSODs are driver/software issues, not hardware failures. Say so.
- Always get the user's consent before changing system settings
  (e.g. `powercfg /h off`). Explain what the command does.
- Run commands as Administrator. The bash tool here runs as admin already.

## Step 1 - Confirm the crash from Event Viewer

Get the BugCheck record (WER-SystemErrorReporting, Event ID 1001) and the
Kernel-Power Event ID 41 (unexpected reboot). The 1001 message contains the
bugcheck code, arguments, and the minidump path.

```powershell
Get-WinEvent -FilterHashtable @{LogName='System'; Id=1001} -MaxEvents 5 |
  Format-List TimeCreated, ProviderName, Message
```

Note: on Chinese Windows, `Message` may display garbled (GBK vs UTF-8). Do NOT
trust garbled text; rely on the numbers (bugcheck code like `0x0000009f`, dump
paths like `C:\windows\Minidump\xxx.dmp`) which remain readable.

Also list recent critical events and the Kernel-Power 41 history:

```powershell
Get-WinEvent -FilterHashtable @{LogName='System'; Level=1} -MaxEvents 10 |
  Format-List TimeCreated, Id, ProviderName
```

## Step 2 - Find the minidump

```powershell
Get-ChildItem C:\Windows\Minidump | Sort-Object LastWriteTime -Descending |
  Select-Object Name, LastWriteTime, Length
```

Prefer the newest `.dmp` matching the crash time. Also check for a full dump:
`C:\Windows\MEMORY.DMP`.

**If a matching dump is found, IMMEDIATELY copy it to a safe working location**
(e.g. `Copy-Item <dump> $env:TEMP\bsod\`) and analyze the copy. Storage Sense /
WER cleanup can delete dumps from `C:\Windows\Minidump` at any time — including
between "found it" and "analyzed it" — leaving only the path in the event log.

**The folder can be EMPTY** — Windows (Storage Sense / WER cleanup) deletes old
minidumps, and a `Get-ChildItem` may return nothing even though the folder
exists. If empty:

1. Check WER archives: `C:\ProgramData\Microsoft\Windows\WER\ReportArchive`
   (and `ReportQueue`) for `.dmp` files, newest first.
2. Fall back to Event-log-only diagnosis (Step 1): BugCheck 1001 still gives
   the stop code, parameters, and dump path even after the file is gone.
3. Do NOT treat missing dumps as "no crash" — always report what the event log
   shows. Note: `C:\Windows\LiveKernelReports` contains live kernel report dumps
   (watchdog/DPC), NOT bugcheck minidumps — do not analyze those as if they
   were BSOD dumps.

## Step 3 - Locate (or install) a debugger

Prefer the classic Windows SDK Debugging Tools first:

- `C:\Program Files (x86)\Windows Kits\10\Debuggers\x64\kd.exe` (KD),
  `cdb.exe` (CDB), `windbg.exe`
- Install via "Windows SDK" → feature "Debugging Tools for Windows", or the
  Microsoft Store "WinDbg" app.

If neither exists, tell the user a debugger is required and offer to install
it. With the user's consent, auto-install the Store WinDbg via winget:

```powershell
winget install --id Microsoft.WinDbg -e --accept-package-agreements --accept-source-agreements
```

After installing, re-run detection and continue with the Store-version flow
below. If winget is unavailable or the user declines, give them the Microsoft
Store link (<https://apps.microsoft.com/detail/9pgj2n31m8w8c>), wait for them
to confirm installation, then re-run detection. NEVER download debugger
binaries from third-party mirrors.

If only the Store version exists:

```powershell
Get-AppxPackage -Name '*WinDbg*' | Format-List Name, Version, InstallLocation
```

Store WinDbg provides native execution aliases in `%LOCALAPPDATA%\Microsoft\WindowsApps\`:
- `cdbX64.exe` (command-line debugger, run directly without elevation or copying)
- `WinDbgX.exe` (GUI)

Do **NOT** copy or dump binaries out of `WindowsApps` or use temp folders. Run the alias directly:

```powershell
& "$env:LOCALAPPDATA\Microsoft\WindowsApps\cdbX64.exe" -version
```

## Step 4 - Analyze the dump

Use CDB with the Microsoft symbol server. **Feed commands via stdin** (one per
line) instead of `-c "cmd; cmd"` — PowerShell's Start-Process and even direct
pipes mangle semicolon-separated `-c` strings, which is a common failure mode.

```powershell
$cmds = ".extpath C:\Users\...\temp\windbg\winext`n!analyze -v`nq"
$cmds | & 'C:\Users\...\temp\windbg\cdb.exe' -z 'C:\Windows\Minidump\xxx.dmp' `
  -y 'srv*C:\Users\...\temp\sym*https://msdl.microsoft.com/download/symbols' `
  -logo 'C:\Users\...\temp\analyze.log' 2>&1
```

- First run downloads symbols (can take 1-5 min). Use a `-logo` file and check
  size/progress; do not assume it hung. Symbol cache makes reruns fast.
- `!analyze -v` gives `BUGCHECK_CODE`, `FAILURE_BUCKET_ID`, `IMAGE_NAME`,
  `STACK_TEXT`, `FAULTING_MODULE`.

## Step 5 - Interpret the result

Key fields from `!analyze -v`:

| Field | Meaning |
|---|---|
| `BUGCHECK_CODE` | The stop code (e.g. `9f`) |
| `BUGCHECK_P1..P4` | Parameters; P1=3 for 0x9F means "device blocked a power IRP too long" |
| `FAILURE_BUCKET_ID` | e.g. `0x9F_3_ACPI_IMAGE_pci.sys` |
| `IMAGE_NAME` / `MODULE_NAME` | Suspected module |
| `STACK_TEXT` | Call stack; `nt!PopIrpWatchdogBugcheck` confirms 0x9F watchdog |

For `0x9F` (DRIVER_POWER_STATE_FAILURE) with P1=3, identify the real culprit by
resolving the blocked IRP (P4) and the device object (P2) — this reveals the
function driver, which is often NOT the module in IMAGE_NAME:

```
!devobj <P2>
!irp <P4>
```

The `!irp` output shows the IRP stack with `\Driver\...` names — the driver
that did not complete is the one whose stack entry shows `pending` at the
current (>) stack location. (Caveat: Windows "kernel generated triage dumps"
contain only registers + stack, so deep memory walks like `!devobj`/`!irp` may
be unavailable — state this limitation honestly in the report.)

### Common bugcheck quick reference

| Code | Name | Typical cause |
|---|---|---|
| 0x9F | DRIVER_POWER_STATE_FAILURE | Driver didn't finish a power IRP (sleep/wake/device idle) |
| 0x124 | WHEA_UNCORRECTABLE_ERROR | Hardware-level: CPU/GPU/memory/PSU stress (can be transient) |
| 0x133 | DPC_WATCHDOG_VIOLATION | Driver stuck in DPC, often storage/GPU/network |
| 0x50 | PAGE_FAULT_IN_NONPAGED_AREA | Memory corruption, bad RAM or driver |
| 0x3B | SYSTEM_SERVICE_EXCEPTION | Usually driver bug |
| 0x1E | KMODE_EXCEPTION_NOT_HANDLED | Driver bug |
| 0x7E | SYSTEM_THREAD_EXCEPTION_NOT_HANDLED | Driver bug |
| 0xC2 | BAD_POOL_CALLER | Driver misuse of pool |
| 0xA | IRQL_NOT_LESS_OR_EQUAL | Driver bug |
| 0xEF | CRITICAL_PROCESS_DIED | System component crash |

## Step 6 - Report + recommendations

Report (plain language, user-friendly):
1. Crash date/time and bugcheck code (confirmed from Event log + dump).
2. Root cause in plain words: what device/driver, what it was doing.
3. Explicit reassurance when it is NOT hardware (common for 0x9F).
4. History: list past crashes (repeat codes signal a pattern).

Recommendations (escalating):
1. Software fix: update the suspect driver (GPU/chipset/network), Windows
   Update, or remove/uninstall the offending third-party software (remote
   desktop, anti-cheat, virtual display drivers).
2. Power mitigation for 0x9F: `powercfg /h off` (disable hibernation, admin
   CMD) AND set Sleep/Hibernate to "Never" in power plan. Explain that
   disabling Fast Startup alone is NOT equivalent.
3. Hardware signal only for 0x124/WHEA: suggest `mdsched` memory test, temps,
   PSU. Never claim hardware is broken from a single event.
4. Offer to execute safe mitigations, with consent.
5. Dump preservation: offer to exclude `C:\Windows\Minidump` from Storage
   Sense cleanup (Settings → System → Storage → Storage sense, or via
   `Cleanmgr` / registry), so future dumps survive long enough to analyze.

## Example run (real case, 2026-09-07)

Full write-up: [`examples/2026-09-07-0x9F-iaLPSS2_I2C.md`](examples/2026-09-07-0x9F-iaLPSS2_I2C.md).

- Event: BugCheck 1001, `0x0000009f` `(0x3, 0xffffb507a20c2360, ...)`,
  dump `090726-7843-01.dmp`.
- `!analyze -v` → `FAILURE_BUCKET_ID: 0x9F_3_ACPI_IMAGE_pci.sys`,
  stack `nt!PopIrpWatchdogBugcheck`.
- `!devobj P2` + `!irp P4` → blocked IRP_MN_SET_POWER, current stack entry
  `pending` at `\Driver\ACPI`, lower stack `\Driver\iaLPSS2_I2C_ADL`
  (Intel chipset I2C controller).
- Conclusion: device power-transition timeout during normal use (device idle
  D-state transition), driver-level issue, not GPU/PSU hardware failure.
- Mitigation: `powercfg /h off` + sleep "Never" + update Intel chipset driver.
