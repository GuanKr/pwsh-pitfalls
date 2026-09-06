# Trigger eval — 2026-09-06 (description optimization loop, round 1)

Method: 14 queries x 3 independent judge subagents (A/B/C), verdict from description text only.
Result: **42/42 unanimous, 0 misses, 0 false-triggers.** No description change required.

| ID | Query gist | Expected | A | B | C |
|----|-----------|----------|---|---|---|
| T1 | Select-String -Recurse error, recursive log search | TRIGGER | TRIGGER | TRIGGER | TRIGGER |
| T2 | Pasted grep/head pipeline, no conversion asked | TRIGGER | TRIGGER | TRIGGER | TRIGGER |
| T3 | Delete .tmp under space-containing dir, confirm first | TRIGGER | TRIGGER | TRIGGER | TRIGGER |
| T4 | .ps1 prints question marks on one machine | TRIGGER | TRIGGER | TRIGGER | TRIGGER |
| T5 | prog < input.txt, reserved-for-future-use error | TRIGGER | TRIGGER | TRIGGER | TRIGGER |
| T6 | vendor.exe -h192.168.1.10 args split apart | TRIGGER | TRIGGER | TRIGGER | TRIGGER |
| T7 | powershell.exe CJK mojibake, pwsh fine | TRIGGER | TRIGGER | TRIGGER | TRIGGER |
| F1 | git status | NO | NO | NO | NO |
| F2 | node version check | NO | NO | NO | NO |
| F3 | list directory | NO | NO | NO | NO |
| F4 | write Python batch-rename script | NO | NO | NO | NO |
| F5 | Get-Service Spooler status | NO | NO | NO | NO |
| F6 | how to configure PATH on Windows | NO | NO | NO | NO |
| F7 | count pwsh processes | NO | NO | NO | NO |

Borderline notes (all judges flagged independently): F4 flips to TRIGGER once deletion executes;
F6 flips on pasted errors/registry edits; T6's quoting-scope wording is the thinnest match.
Watch these three in real-world trials; they are the candidates for round 2.
