# bash-to-pwsh translation table

Read this file only when converting a bash pipeline to PowerShell. Rows marked (M) were verified by local measurement; the base table shape is adapted from powershell-windows-cli-agent-skill.

## Files

| bash | pwsh |
|---|---|
| ls -la | Get-ChildItem -Force |
| ls -R | Get-ChildItem -Recurse |
| cp / mv / rm / mkdir -p | Copy-Item / Move-Item / Remove-Item / New-Item -ItemType Directory -Force |
| find DIR -name LOG | Get-ChildItem DIR -Filter LOG -Recurse -File (M: bare find is Windows find.exe) |
| touch f (keep content, update timestamp) | if (Test-Path f) { (Get-Item f).LastWriteTime = Get-Date } else { New-Item -ItemType File f } (New-Item -Force alone TRUNCATES existing files) |

## Text

| bash | pwsh |
|---|---|
| cat f | Get-Content f |
| grep PAT f | Select-String -Path f -Pattern PAT |
| grep -r PAT dir | Get-ChildItem dir -Recurse -File then pipe to Select-String (M: Select-String has no -Recurse) |
| sed -i s/a/b/g f | (Get-Content f) -replace 'a','b' piped to Set-Content f -Encoding UTF8 |
| head/tail -n 10 f | Get-Content f -TotalCount 10 / -Tail 10 |
| sort then uniq | Sort-Object -Unique |
| wc -l f | (Get-Content f).Count (off by one vs wc on files without trailing newline: pwsh counts the last partial line) |
## Processes, services, network

| bash | pwsh |
|---|---|
| ps aux / kill PID | Get-Process / Stop-Process -Id PID -WhatIf |
| systemctl status/start/stop S | Get/Start/Stop-Service -Name S (destructive ones: -WhatIf first) |
| curl URL | Invoke-RestMethod -Uri URL (plain curl also ships with Windows) |
| ping -c4 H | Test-Connection -ComputerName H -Count 4 |
| awk '{print $1}' | Get-Content f piped to ForEach-Object { ($_ -split '\s+')[0] } |

| xargs -I{} cmd {} | pipe into ForEach-Object { cmd $_ } |
