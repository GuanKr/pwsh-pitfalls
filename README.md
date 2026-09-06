# pwsh-pitfalls

Windows PowerShell failure-repair skill for AI coding agents. Zero preflight, zero behavior tax.

General-purpose agents are trained mostly on Linux/bash and fail on Windows in predictable ways:
wrong Unix idioms (`Select-String -Recurse`, bare `find`, `prog < file`), Boolean `$?`,
`-Path` vs `-LiteralPath`, console-codepage encoding, one-shot `-ExecutionPolicy Bypass`,
and native-argument mangling (`-h127.0.0.1` arriving split in two). This skill encodes each
pitfall as symptom-plus-fix, verified by local measurement on Windows 10 + PowerShell 7.

## Design

- **Reactive, not ambient.** Triggers on failed pwsh commands, destructive/recursive file ops,
  and bash-to-pwsh translation. Routine commands that already work explicitly do NOT trigger.
- **No scripts, no gates.** Pure knowledge: read the matching trap, apply, run once.
  (Deliberate contrast to preflight-script skills: every check they mandate costs a tool call.)
- **Measured, not folklore.** Latency and parsing claims carry local benchmark numbers.

## Install

With the skills CLI (Claude Code / Codex compatible):

```sh
npx skills add <you>/pwsh-pitfalls
```

Manual: copy `SKILL.md` + `references/` into your agent skills directory
(`~/.claude/skills/pwsh-pitfalls/` for Claude Code) and restart the session.

## Layout

```
SKILL.md                      # 7 traps + trigger/non-trigger contract (<5 KB)
references/bash-to-pwsh.md    # translation table, read only when converting pipelines
evals/evals.json              # 4 trigger/negative test prompts
```

## Acknowledgments

Built by absorbing `agent-shells/powershell-skills` (risk tiers, `-LiteralPath` discipline)
and `UncertaintyDeterminesYou4ndMe/powershell-windows-cli-agent-skill` (Rosetta table,
safety patterns), following the official `anthropics/skills` skill-creator methodology.

## License

MIT — see LICENSE. Copyright holder is set at publish time.
