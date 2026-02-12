---
name: show
description: Show current permission settings across all scopes with a comparison table
disable-model-invocation: true
allowed-tools: Read, Glob, Bash
---

# grantly:show — Permission Settings

Display all permission rules currently applied to the project, comparing across scopes.

## Processing Flow

### Step 1: Read Settings Files

Read the following 3 files with Read (treat as empty if they don't exist):

1. `~/.claude/settings.json` — user scope
2. `.claude/settings.json` — project scope (relative to current directory)
3. `.claude/settings.local.json` — local scope (relative to current directory)

### Step 2: Analyze Rules

Merge all rules (allow/deny/ask) from all scopes into a unique pattern list and map which scopes each rule exists in.

### Step 3: Output Results

```
## grantly:show — Permission Settings

                              user    project  local
  Read                         ✓
  Grep                         ✓
  Glob                         ✓
  Bash(git status:*)           ✓                 ✓
  Bash(git diff:*)             ✓                 ✓
  Bash(npm run test:*)                           ✓
  Bash(npm run lint:*)                           ✓
  Edit(src/**)                                   ✓
  Bash(rm -rf:*)                        [deny]

---

Legend: ✓ = allow, [deny] = deny, [ask] = ask

user only:   3 items (Read, Grep, Glob)
local only:  3 items (Bash(npm run test:*), Bash(npm run lint:*), Edit(src/**))
duplicates:  2 items (Bash(git status:*), Bash(git diff:*))

Total: allow 9 / deny 1 / ask 0
```

Display rules:
- Show each pattern's presence per scope with `✓` / `[deny]` / `[ask]`
- Sort patterns alphabetically
- Display a summary footer:
  - Count of rules that exist in only one specific scope
  - Count of rules that exist in multiple scopes
  - Grand total across all scopes (allow / deny / ask)
- If a settings file does not exist, omit that scope's column
- If no rules exist at all, display: "No permission rules configured."
- Read-only — no settings are modified
