---
name: pwsh-pitfalls
description: Repairs failed Windows PowerShell (pwsh) commands and translates bash idioms to pwsh. Triggers when a pwsh command just failed on Windows, before destructive or recursive file operations, when converting grep/sed/awk/find/head pipelines to PowerShell (even a pasted bash pipeline nobody asked to convert), or when quoting, paths, or encoding look suspicious. Do NOT use for routine commands that already work.
---

# pwsh-pitfalls: Windows PowerShell failure repair, zero preflight

No scripts to run, no checklist gates, no mandatory pre-checks. Read the matching trap, apply the fix, run once. Routine commands that already work must NOT trigger extra reads or checks.

## Rule 0: stay in pwsh, escalate to bash only for pipelines

Each harness call already spawns pwsh. A nested bash call adds about 100ms spawn tax and a quoting layer, and measured slower end to end. Prefer native cmdlets; reach for bash only for a grep/sed/awk/find pipeline you cannot express in one pwsh line.
Bash escape hatch, only safe shape is outer SINGLE quotes, never double:

    bash --noprofile --norc -c 'grep -r --include="*.md" "pattern" . 2>/dev/null | head -n 20'

Outer double quotes let the parent pwsh expand $-vars first and bash then eats the backslashes: echo $HOME arrives mangled (backslashes eaten, e.g. C:UsersYOU), echo $? arrives as True. If the inner command contains a single quote, rewrite the command instead of stacking escapes.

## Trap 1: Unix tools are missing, and find is an impostor

pwsh has no grep/sed/awk/head/tail/wc. Worse: find resolves to Windows find.exe, a text search, NOT file search. Select-String has NO Recurse parameter (it only scans content streams handed to it), enumerate first:

    Get-ChildItem -Recurse -File | Select-String -Pattern 'error' | Select-Object -First 20
    Select-String emits MatchInfo objects, not plain text: append Select-Object -ExpandProperty Line for text
    Get-Content file.log -TotalCount 10
    Get-Content file.log -Tail 10
    (Get-Content file.log).Count
    Get-Content f.txt | Sort-Object -Unique
    (Get-Content f.txt) -replace 'foo','bar' | Set-Content f.txt -Encoding UTF8

Redirection is half-missing: `>`/`>>` exist, but `<` is a hard ParserError ("reserved for future use"). Feed stdin through the pipe instead:
    Get-Content query.sql | prog          # not: prog < query.sql

Full table: read references/bash-to-pwsh.md only when translating a pipeline.

## Trap 2: exit codes have two type systems

- $? is a Boolean True/False, not a number. Never compare it with -eq 1.
- Native programs report via $LASTEXITCODE; cmdlets leave it untouched.
- Test failure as: if (-not $?) { ... } for cmdlets, if ($LASTEXITCODE -ne 0) { ... } for external programs.

## Trap 3: paths with spaces, brackets, or non-ASCII

- Exact paths: always use -LiteralPath, because square brackets are wildcards under -Path and a path like log[1].txt would silently match nothing.
- Compose paths with Join-Path, never string concatenation.
- Remove-Item -Recurse -Force on a resolved path only: print the resolved absolute path first, then delete.

## Trap 4: encoding lies at the boundary

PS7 pipeline strings are UTF-8, but external .exe output is decoded with the console code page. If CJK output garbles, suspect the boundary before suspecting the data. Use Set-Content -Encoding UTF8 on write. To force UTF-8 decoding of external program output in-session, set the console output encoding to UTF8 before running it.

Input side: .ps1 source encoding. powershell.exe (5.1) parses script bytes with the system ANSI codepage, so BOM-less UTF-8 containing CJK arrives as mojibake (verified: identical file prints garbage under 5.1, correct under pwsh 7). Fixes, version-aware: target 5.1 -> save WITH BOM (pwsh 7: Set-Content -Encoding utf8BOM; warning: 5.1 has no utf8BOM name, its -Encoding UTF8 already writes a BOM); or keep scripts ASCII-only; or confirm the runner is pwsh 7, which reads BOM-less UTF-8 natively.
## Trap 5: execution policy, one-shot only

"Running scripts is disabled on this system" means: add -ExecutionPolicy Bypass to that single invocation only, which leaves no persistent security change behind.

    pwsh -ExecutionPolicy Bypass -File .\script.ps1

Never Set-ExecutionPolicy machine-wide, never Unrestricted as a default. CurrentUser RemoteSigned is the ceiling for persistent changes.

## Trap 6: destructive and state-changing commands

- Preview first: Remove-Item -Recurse dir -WhatIf, Stop-Process -Name x -WhatIf. -Recurse combined with wildcards gets -WhatIf first, no exceptions.
- State-changing steps get -ErrorAction Stop or try/catch. Do not repeat a failed shape unchanged: re-read the error and change one thing per retry.
- Never Invoke-Expression on untrusted input; pass arguments as arrays to Start-Process -ArgumentList.
- Saved .ps1 files use full cmdlet names, no ls/?/% aliases. Compare with null on the LEFT: $null -eq $x.
- Write-Output returns data down the pipeline; Write-Host is console-only display.

## Trap 7: compact dotted flags get split for native programs

PowerShell re-parses `-flagvalue` tokens: measured `-h127.0.0.1` arriving as two arguments (`-h127`, `.0.0.1`), while `-h 127.0.0.1`, `--host=127.0.0.1`, and quoted `'-h127.0.0.1'` all arrive intact. Prefer space-separated or `=` form for native EXEs; quote compact flags only when the tool insists on them.

## Explicit non-triggers (behavior-tax guard)

pwd, dir listings, cat, git status, version checks, any command that succeeded last time: run directly with no extra reads. If this skill was already read once in the session, do not re-read it for the same trap twice.

