# bsod-diagnostic-skill

English | [简体中文](README.zh-CN.md)

**Install** — paste this to your AI agent:

```text
Install the skill from https://github.com/itwxb/bsod-diagnostic-skill into your skills folder and tell me how to use it.
```

An AI agent skill for diagnosing Windows blue-screen (BSOD) crashes using the
standard Microsoft debugging workflow — Event Viewer → minidump → WinDbg/cdb →
`!analyze -v` with the Microsoft symbol server — and turning the raw output into
a plain-language explanation and actionable fix list.

## What it does

- Confirms the crash from the System event log (BugCheck 1001 / Kernel-Power 41)
- Locates the matching minidump (`C:\Windows\Minidump`), with fallbacks when
  Windows cleanup has already deleted it
- Finds a debugger (Windows SDK / Store WinDbg) and solves the common
  WindowsApps permission + extension-DLL pitfalls automatically
- Runs `!analyze -v` with `msdl.microsoft.com` symbol server and interprets
  `BUGCHECK_CODE`, `FAILURE_BUCKET_ID`, `STACK_TEXT`
- Digs into 0x9F power IRP crashes with `!devobj` / `!irp` to find the real
  offending driver (not just the module name in the bucket)
- Explains results in plain language, with history of past crashes and
  step-by-step fix recommendations

## Installation

### opencode

Copy the `bsod-diagnostic-skill` folder to one of:

- Global: `~/.config/opencode/skills/bsod-diagnostic-skill/`
- Project: `.opencode/skills/bsod-diagnostic-skill/`

Or publish it and register via URL in `opencode.json`:

```json
{
  "skills": {
    "urls": ["https://example.com/skills/"]
  }
}
```

### Claude Code / other Anthropic-compatible agents

```powershell
Copy-Item -Recurse .\bsod-diagnostic-skill "$env:USERPROFILE\.claude\skills\"
```

Restart the agent after installing. A skill is just a `SKILL.md` file inside a
folder named after it, so any SKILL.md-compatible agent can load it.

## Usage

Just describe what happened — e.g. "I just got a blue screen, help me figure
out why" — and the skill walks the agent through: confirming the crash time,
analyzing the dump, and giving you a plain-language answer with fixes.

If the agent doesn't trigger the skill automatically, invoke it explicitly:

```text
Use the bsod-diagnostic-skill to analyze my blue screen.
```

## Requirements

- Windows 10/11, PowerShell 5.1+ (run as Administrator)
- First analysis needs internet access to download Microsoft symbols
  (msdl.microsoft.com); allow 1–5 minutes
- A debugger is recommended: Windows SDK "Debugging Tools for Windows" or the
  Microsoft Store "WinDbg" app (the skill finds it automatically, and can also
  extract a working cdb.exe from the Store version)

## Notes & limitations

- **No data leaves your machine.** The skill only reads local event logs and
  dump files; it does not upload them anywhere.
- Windows "kernel generated triage dumps" contain only registers and stacks,
  so some deep memory analysis (`!devobj`/`!irp`) may be unavailable; the skill
  reports this honestly instead of guessing.
- A single BSOD is usually a **driver/software issue, not hardware failure**.
  Hardware-level signals (e.g. 0x124 WHEA) are treated with extra caution and
  never used to claim hardware is broken from one event.

## License

MIT — free to use, modify, and redistribute. See [LICENSE](LICENSE).

## Contributing

PRs welcome. When adding a new case to [`examples/`](examples/), include the
bugcheck code, the `!analyze -v` bucket ID, and the `!irp` stack.
