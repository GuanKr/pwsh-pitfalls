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

`-Filter` uses filesystem wildcards where `?` is exactly one character, and a non-matching pattern returns zero results **with no error**: `-Filter '00000?.up.sql'` silently misses `000001_initial_platform.up.sql` because position 7 is `_`, not `.`. For name patterns, filter by regex instead: `Get-ChildItem $d -Filter '*.sql' | Where-Object { $_.Name -match '^\d{6}_' }`.

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

Write side is a second, independent boundary, and it is the one that bites hardest — but not for the reason most assume. In **PS7 `$OutputEncoding` already defaults to UTF-8**, so `native > file` preserves CJK correctly (verified on 7.6.6: UTF-8 source through `>` comes back byte-identical). The real failure is upstream of PowerShell: **the native tool's own default encoding silently transcodes your data**. Measured: `mysql -e "SHOW CREATE TABLE ..."` without `--default-character-set=utf8mb4` on a Chinese Windows box reports `@@character_set_client = gbk` and returns `DEFAULT '在产'` as `DEFAULT '?ڲ?'` — two bytes destroyed, no error, valid-looking SQL. The corruption entered the database because that string was fed into a generated `ALTER`.

Rule: **when a native CLI round-trips non-ASCII, set its encoding flag explicitly; never let its default stand.** `--default-character-set=utf8mb4` (mysql), `-c i18n.log_output_encoding=UTF-8` (git), and so on. `[Console]::OutputEncoding` does nothing for this — it governs the shell, not the child. Corollary: suspect the tool's default before suspecting the shell, because the shell is usually already correct.

When you need bytes to survive untouched regardless of what any layer assumes — dumps, binaries, checksums, anything that must match an external checksum — skip text entirely:

    $p = [Diagnostics.ProcessStartInfo]::new(); $p.FileName = 'mysqldump.exe'
    foreach ($a in @('-uroot','--host=127.0.0.1','--default-character-set=utf8mb4','somedb')) { $p.ArgumentList.Add($a) }
    $p.RedirectStandardOutput = $true; $p.RedirectStandardError = $true; $p.UseShellExecute = $false
    $pr = [Diagnostics.Process]::Start($p)
    $fs = [IO.File]::Create($out); $pr.StandardOutput.BaseStream.CopyTo($fs); $fs.Close()
    $pr.WaitForExit()

Shorter form for the same bytes: `Start-Process -FilePath x -ArgumentList @(...) -RedirectStandardOutput $out -Wait -NoNewWindow`. When a generated file's non-ASCII looks wrong, diff it against a known-good copy before shipping — encoding damage is silent and propagates straight into whatever consumes the file.

Note for Windows PowerShell 5.1, which behaves differently here: its `$OutputEncoding` defaults to ASCII, so `>` and `|` do corrupt CJK there, and the raw-bytes form above is the fix rather than an optimization.

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

## Trap 8: the characters you typed are not the characters that ran

Three silent-identity breaks. All three produce an error that points somewhere other than the cause.

**Curly quotes are string delimiters.** `“ ”` and `‘ ’` are legal PowerShell string delimiters, so a `“` inside a double-quoted string terminates it early. Measured: one Chinese header line turned into `ParserError ... 方法调用中缺少 ")"`, reported at an unrelated line, with the real line several lines up. In any script containing CJK, write `【 】` or plain quotes inside double-quoted strings. To locate it without guessing, parse the file first:

    $e=$null; [void][System.Management.Automation.Language.Parser]::ParseFile($f,[ref]$null,[ref]$e); $e | Select-Object -First 3

**Backtick is the escape char inside double quotes.** Writing `` `version` `` yields `ersion` (the `v` is eaten); a literal backtick needs ``` `` ```. This bites when generating SQL identifiers or Markdown. Rule: **any string containing a backtick is a single-quoted string**, never double-quoted. Same trap inverted: `` "`$var" `` escapes the dollar and yields the literal text `$var` — build such values by concatenation instead, `'x' + $var`.

**Variable names are case-insensitive.** `$RB` and `$rb` are the same variable. Assign an encoding object to `$RB`, later assign a file's text to `$rb`, and the first is gone — the follow-on call then fails with `找不到"ReadAllText"的参数计数为"2"的重载`, which names neither variable. Name such values unambiguously (`$encUtf8`, not `$RB`), and when a method-call overload error appears, check whether an argument was silently overwritten before checking the method signature.

## Explicit non-triggers (behavior-tax guard)

pwd, dir listings, cat, git status, version checks, any command that succeeded last time: run directly with no extra reads. If this skill was already read once in the session, do not re-read it for the same trap twice.

