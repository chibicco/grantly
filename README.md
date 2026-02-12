# grantly

Claude Code plugin for smarter permission management.

Analyzes your session history and suggests **per-command** permission rules — the right granularity between "Always allow" (too specific) and hand-written wildcards (too broad).

```
"Always allow":  Bash(git log -5)      <- breaks on different args
Hand-written:    Bash(git *:*)         <- allows git push too
grantly:         Bash(git log:*)       <- per-command, args free
                 Bash(git status:*)
                 Bash(git diff:*)
                 (git push → high-risk warning)
```

## Prerequisites

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) v1.0.33 or later
- Python 3 (used internally for session history parsing)

## Install

```bash
# Add marketplace
/plugin marketplace add chibicco/grantly

# Install plugin
/plugin install grantly@grantly
```

For local development:

```bash
claude --plugin-dir /path/to/grantly
```

## Usage

All skills respond in English by default. To get output in another language, pass instructions as an argument:

```
/grantly:suggest 日本語で応答してください。
```

## Skills

### `/grantly:suggest`

Scan recent sessions and suggest permission candidates.

```
## grantly:suggest — Permission Candidates

Project: my-webapp
Analyzed: last 5 sessions (2026-01-29 ~ 2026-02-12)

### Recommended (Low Risk)
  1. Read                        45 uses / 5 sessions: 5
  2. Bash(git status:*)          15 uses / 5 sessions: 5
  3. Bash(git diff:*)            12 uses / 5 sessions: 5

### Review (Medium Risk)
  4. Bash(npm run test:*)        14 uses / 5 sessions: 4
  5. Bash(npm run lint:*)         7 uses / 5 sessions: 3

### Caution (High Risk)
  6. Bash(git push:*)             3 uses / 5 sessions: 3

Select numbers to add (e.g., 1-3, 4)
```

Pick numbers, choose a scope (user / project / local), done.

### `/grantly:show`

Display current permission rules across all scopes with a comparison table.

```
## grantly:show — Permission Settings

                              user    project  local
  Read                         ✓
  Bash(git status:*)           ✓                 ✓
  Bash(npm run test:*)                           ✓
  Edit(src/**)                                   ✓
  Bash(rm -rf:*)                        [deny]

Legend: ✓ = allow, [deny] = deny, [ask] = ask

user only:   1 item (Read)
duplicates:  1 item (Bash(git status:*))

Total: allow 5 / deny 1 / ask 0
```

### `/grantly:tidy`

Detect and clean up redundant rules — duplicates, subset overlaps, allow/deny conflicts.

```
## grantly:tidy — Permission Cleanup Candidates

Issues detected: 2

### Cross-Scope Duplicates (1 item)

  1. `Bash(git status:*)` — already allowed in user scope. Can be removed from local scope

### Subset Redundancy (1 item)

  2. `Read(src/**)` — unnecessary because `Read` (full access) exists

Select numbers to fix (e.g., 1-2)
Fix all: a / Skip: s
```

## Project structure

```
grantly/
├── .claude-plugin/
│   ├── plugin.json
│   └── marketplace.json
├── commands/          ← Thin wrappers that delegate to skills/
│   ├── suggest.md
│   ├── show.md
│   └── tidy.md
└── skills/            ← Actual skill implementations
    ├── suggest/
    │   └── SKILL.md
    ├── show/
    │   └── SKILL.md
    └── tidy/
        └── SKILL.md
```

`commands/` exists as a workaround for [a known bug](https://github.com/anthropics/claude-code/issues/22517) where `skills/` autocomplete shows names without the plugin namespace prefix (e.g. `/suggest` instead of `/grantly:suggest`). Each command file simply delegates to the corresponding skill via `/skill <name>`. Once the bug is fixed, `commands/` can be removed and `skills/` alone will be sufficient.

## How it works

1. Reads `~/.claude/projects/<project-dir>/*.jsonl` (last 5 sessions)
2. Extracts `tool_use` entries and groups by command prefix
3. Compares against existing `settings.json` rules
4. Presents new candidates with risk levels
5. Writes selected rules to your chosen settings file

No external dependencies. No data leaves your machine.

## License

[MIT](./LICENSE)
