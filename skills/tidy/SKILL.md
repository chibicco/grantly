---
name: tidy
description: Clean up redundant and duplicate permission rules
disable-model-invocation: true
allowed-tools: Read, Glob, Bash, Write
---

# grantly:tidy — Permission Cleanup

Detect redundant or conflicting rules in existing permission settings and propose cleanup actions.

## Detection Targets

### 1. Exact Duplicates

Identical rules existing within the same scope or across scopes.

Example:
```
user:  allow: ["Read"]
local: allow: ["Read"]        ← already covered by user scope
```

### 2. Subset Redundancy

A narrower rule exists alongside a broader rule that already covers it.

Example:
```
allow: ["Read", "Read(src/**)"]       ← Read grants full access, so Read(src/**) is unnecessary
allow: ["Bash(git *:*)", "Bash(git status:*)"]  ← git * grants full access, so git status is unnecessary
```

### 3. Allow/Deny Conflicts

The same pattern exists in both allow and deny.

Example:
```
allow: ["Bash(rm:*)"]
deny:  ["Bash(rm:*)"]         ← conflict (deny takes precedence, but intent is unclear)
```

### 4. Cross-Scope Duplicates

A rule permitted in a higher scope (user) is also duplicated in a lower scope (project/local).

Scope priority: user → project → local (user has the broadest reach)

## Processing Flow

### Step 1: Read Settings Files

Read the following 3 files with Read (treat as empty if they don't exist):

1. `~/.claude/settings.json` — user scope
2. `.claude/settings.json` — project scope
3. `.claude/settings.local.json` — local scope

### Step 2: Detect Issues

Detect all 4 issue types above and compile a list.

### Step 3: Output Results

```
## grantly:tidy — Permission Cleanup Candidates

Issues detected: <N>

---

### Cross-Scope Duplicates (<N> items)

  1. `Read` — already allowed in user scope. Can be removed from local scope
  2. `Bash(git status:*)` — already allowed in user scope. Can be removed from local scope

### Subset Redundancy (<N> items)

  3. `Read(src/**)` — unnecessary because `Read` (full access) exists
  4. `Bash(git status:*)` — unnecessary because `Bash(git *:*)` exists

### Allow/Deny Conflicts (<N> items)

  5. `Bash(rm:*)` — exists in both allow and deny (deny takes precedence)

---

Select numbers to fix (e.g., 1-3, 5)
Fix all: a / Skip: s
```

- If 0 issues are found:
  ```
  No redundant or conflicting rules found.
  Your current settings are clean.
  ```

### Step 4: User Selection and Application

Wait for user input and process as follows:

1. **Parse number selection:**
   - `1,2,3` → individual selection
   - `1-4` → range selection
   - `1-3, 5` → mixed
   - `a` → select all items
   - `s` → skip, display "No changes made." and stop

2. **Confirm the changes:**
   ```
   The following rules will be removed:

     local scope (.claude/settings.local.json):
       allow:
         - Read
         - Bash(git status:*)

   Apply? (y/n) [y]:
   ```

3. For selected issues, remove the target rules from the appropriate settings file
4. Write the updated file with the Write tool
5. Display the result:
   ```
   Cleaned up <N> rule(s) from <file path>.
   ```

## Important Notes

- Only remove rules — never add new ones
- For conflict resolution, default to keeping the deny side and removing the allow side
- Each change only edits the target scope's file (other scopes are not modified)
