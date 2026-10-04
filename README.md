# Windows 10 Task Manager crashes: fixing a malformed Claude Desktop startup entry

A malformed Claude Desktop startup command can cause Windows 10 Task Manager (`Taskmgr.exe`) to crash. Correcting that single registry value resolved the problem in the case documented here, without uninstalling Claude Desktop or reinstalling Windows.

This is an independently confirmed case of an already reported issue, not a claim of a new discovery or a universal fix for all Task Manager crashes. Diagnosis and the write-up were assisted by OpenAI Codex; the computer owner ran the repair and confirmed that the crash stopped.

## Symptoms and tested environment

The owner reported a crash when switching to the Details tab. Other reports describe a crash when expanding Task Manager with **More details** or loading startup information. The visible action does not necessarily identify the background component that failed.

The affected machine ran Windows 10 Pro 22H2, build `19045.6466`. The crashing Task Manager version recorded in Windows Error Reporting was `10.0.19041.6280`.

Repeated crashes had this signature:

```text
Faulting application: Taskmgr.exe
Faulting module:      Taskmgr.exe
Exception code:      0xc0000005
Fault offset:        0x00000000000748a1
```

An access violation code alone does not identify the cause or prove defective RAM. This guide applies when the malformed Claude startup entry below is actually present.

## What was wrong

The registry key was:

```text
HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\Run
```

The value name was `Claude`, with type `REG_SZ`.

Its original content had an extra outer pair of quotes and literal backslash-quote sequences:

```text
"\"C:\Users\<username>\AppData\Local\AnthropicClaude\claude.exe\" --startup"
```

The corrected content was:

```text
"C:\Users\<username>\AppData\Local\AnthropicClaude\claude.exe" --startup
```

These examples show the actual registry text, not JSON or a `.reg` file representation. In the correct command, quotes enclose the executable path and `--startup` is a separate argument. In a serialized file, escaping can be appropriate; here the extra escaping characters were part of the stored command itself.

The string resembles double escaping or double serialization. The available evidence does not establish which installation, update, or other action originally wrote it.

## Evidence from this case

1. Windows Error Reporting repeatedly recorded the same application, exception code, and fault offset.
2. Analysis of a local crash dump using Microsoft public symbols mapped the fault address to `TmGetDetailsFromCrudeCmdLine`, a function whose name indicates parsing a startup command line.
3. Stack memory contained the malformed command pointing to `AnthropicClaude\claude.exe`; candidate return addresses also mapped to startup registry enumeration functions. These candidates were inspected directly, rather than treated as a fully unwound debugger stack trace.
4. The registry export confirmed that the malformed `Claude` value was present.
5. After backing up the startup key and correcting only that value, the owner confirmed that Task Manager no longer crashed.

The registry before/after comparison and successful targeted repair are the strongest practical confirmation. Claude Desktop remained installed, and the corrected value retained its automatic-startup argument.

## Inspect before changing anything

Open **Windows PowerShell** under the Windows account affected by the crash. A different administrator account has a different `HKEY_CURRENT_USER` registry hive.

Run:

```powershell
Get-ItemPropertyValue -LiteralPath 'HKCU:\Software\Microsoft\Windows\CurrentVersion\Run' -Name 'Claude'
```

Compare the result with the examples above. If the value is absent, already correctly quoted, or points to a different installation layout, this specific repair may not apply.

## Back up and repair the matching entry

Close Task Manager before testing the change. The following PowerShell snippet:

- requires the exact malformed pattern described here;
- checks that the expected executable exists;
- backs up only the original `Claude` value and its registry type;
- updates only that value.

It does not require elevation for an ordinary writable per-user Run key. Do not switch accounts to run it.

```powershell
$key = 'HKCU:\Software\Microsoft\Windows\CurrentVersion\Run'
$name = 'Claude'
$exe = Join-Path $env:LOCALAPPDATA 'AnthropicClaude\claude.exe'
$correct = '"' + $exe + '" --startup'
$knownMalformed = '"\"' + $exe + '\" --startup"'
$current = Get-ItemPropertyValue -LiteralPath $key -Name $name -ErrorAction Stop

if ($current -ceq $correct) {
    throw 'The Claude entry is already correctly quoted. No change was made.'
}
if ($current -cne $knownMalformed) {
    throw 'The entry does not match this documented malformed pattern. No change was made.'
}
if (-not (Test-Path -LiteralPath $exe -PathType Leaf)) {
    throw 'The expected Claude executable was not found. No change was made.'
}

$kind = (Get-Item -LiteralPath $key).GetValueKind($name).ToString()
if ($kind -ne 'String') {
    throw 'The registry value is not REG_SZ. No change was made.'
}
$suffix = [guid]::NewGuid().ToString('N').Substring(0, 8)
$backup = Join-Path (Get-Location).Path ("claude-startup-backup-$suffix.clixml")
[pscustomobject]@{
    Key = $key
    Name = $name
    Kind = $kind
    Value = $current
} | Export-Clixml -LiteralPath $backup -ErrorAction Stop

Set-ItemProperty -LiteralPath $key -Name $name -Value $correct -ErrorAction Stop
Write-Host "Corrected Claude startup entry. Backup: $backup"
Get-ItemPropertyValue -LiteralPath $key -Name $name
```

Reopen Task Manager and test the action that previously caused the crash, plus the Startup tab. This case was resolved immediately after the registry correction; no reboot was needed.

The machine-specific repair used in this case had backup and path checks. The generalized snippet above is provided for inspection and reuse; it has not been executed against other people's installations.

## Restore the original value if necessary

Replace the backup path below with the path printed by the repair snippet. Restoring the original malformed value may bring the crash back.

```powershell
$saved = Import-Clixml -LiteralPath '.\claude-startup-backup-REPLACE.clixml'
if ($saved.Key -ne 'HKCU:\Software\Microsoft\Windows\CurrentVersion\Run' -or
    $saved.Name -ne 'Claude' -or $saved.Kind -ne 'String') {
    throw 'Unexpected backup contents. Nothing was restored.'
}
Set-ItemProperty -LiteralPath $saved.Key -Name $saved.Name -Value $saved.Value -ErrorAction Stop
```

## Why the earlier troubleshooting did not resolve it

`DISM /Online /Cleanup-Image /RestoreHealth` completed successfully, and `sfc /scannow` reported no integrity violations. Running Task Manager as administrator and resetting its preferences also did not stop the crashes.

Those steps did not correct the malformed startup registry value. Correcting that value did.

## Prior reports and limits

This behavior was reported before this case:

- [Anthropic GitHub issue #44356](https://github.com/anthropics/claude-code/issues/44356), opened April 6, 2026, reports the same exception code and fault offset associated with Claude Desktop startup registration.
- [Anthropic GitHub issue #48055](https://github.com/anthropics/claude-code/issues/48055), opened April 14, 2026, documents the malformed command and repeated tests: malformed quoting reproduced the crash; normal quoting or removing the value restored Task Manager.
- [Microsoft: Access Violation C0000005](https://learn.microsoft.com/en-us/shows/inside/c0000005) explains the exception category.

The Anthropic reports concern Claude Desktop but were filed in the Claude Code repository and are closed with labels indicating that they are unrelated to Claude Code. Their closed status should not be read as proof that every affected Desktop installation has been fixed.

This guide does not establish the complete affected-version range, the precise code path that originally wrote the registry value, or an official vendor fix version. It also does not establish that every Task Manager access violation has this cause.

If the problem returns after an application update or reinstall, inspect the current value again rather than assuming it is still malformed. Keep crash dumps and complete registry exports private: they can contain unrelated personal information. This write-up contains sanitized evidence and does not include those files.

Case confirmed: October 4, 2026.
