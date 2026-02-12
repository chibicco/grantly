---
name: suggest
description: Analyze session history and suggest permission candidates for settings.json
disable-model-invocation: true
allowed-tools: Read, Glob, Bash, Write
---

# grantly:suggest — Permission Suggestion Skill

You are a skill that assists with Claude Code permission management.
Analyze session history and suggest candidates for `settings.json` `permissions.allow` at **per-command granularity**.

## Core Principle: "Just Right Granularity"

- **Never** suggest overly broad patterns like `Bash(git *:*)`
- Always suggest per-command patterns: `Bash(git log:*)`, `Bash(git status:*)`, `Bash(npm run test:*)`
- Let the user decide on a per-command basis whether to allow each one

## Processing Flow

Execute the following steps in order. Do not display intermediate progress to the user — only show the final result.

### Step 1: Identify the Project

1. Run `pwd` via Bash to get the current working directory's absolute path
2. Replace slashes with hyphens and prepend a hyphen to generate the project directory name
   - Example: `/home/user/projects/my-app` → `-home-user-projects-my-app`
3. Run `ls ~/.claude/projects/<project-dir>/` via Bash to verify the directory exists
4. If it does not exist, display the following and stop:
   ```
   No session history found.
   Run a few sessions with Claude Code first, then try /grantly:suggest again.
   ```

### Step 2: List Sessions

1. Use Glob to find `~/.claude/projects/<project-dir>/*.jsonl`
2. Read the first line of each file with Read and extract the `timestamp` field
3. Sort by `timestamp` in descending order (newest first)
4. Select the most recent 5 sessions for analysis (use fewer if less than 5 exist)
5. If 0 sessions are found, display the following and stop:
   ```
   No sessions found to analyze.
   Run a few sessions with Claude Code first, then try /grantly:suggest again.
   ```

### Step 3: Extract and Aggregate tool_use Entries

Process each session JSONL via Bash. Since JSONLs can be very large, use the following Bash command instead of Read to extract tool_use entries:

```bash
grep '"tool_use"' <file> | python3 -c "
import sys, json
for line in sys.stdin:
    try:
        obj = json.loads(line.strip())
        if obj.get('type') != 'assistant': continue
        for c in obj.get('message',{}).get('content',[]):
            if isinstance(c, dict) and c.get('type') == 'tool_use':
                print(json.dumps({'name': c['name'], 'input': c.get('input',{})}))
    except: pass
"
```

Aggregate the extracted tool_use entries according to the following rules:

#### Bash Command Prefix Extraction

Determine the permission pattern prefix from the command string. **Always aggregate at the per-command level — never roll up into broader patterns.**

1. Get `input.command`
2. If chained with `&&` or `;`, split and process each part individually
3. Ignore `cd ...` (only aggregate the subsequent commands in the chain)
4. Split the command string by whitespace and determine the prefix:

| First Token | Prefix Depth | Examples |
|-------------|-------------|----------|
| `git` | First 2 tokens | `git status`, `git log`, `git push` |
| `npm` + `run` | First 3 tokens | `npm run test`, `npm run lint` |
| `npx` | First 2 tokens | `npx tsc`, `npx prisma` |
| `bundle` + `exec` | First 3 tokens | `bundle exec rspec`, `bundle exec rails` |
| `docker` + `compose` | First 3 tokens | `docker compose up`, `docker compose down` |
| `cargo` | First 2 tokens | `cargo build`, `cargo test` |
| `make` | First 2 tokens | `make build`, `make test` |
| `bin/rails` | First 2 tokens | `bin/rails generate`, `bin/rails console` |
| Other | First token only | `python3`, `pytest`, `node` |

5. Generate a `Bash(<prefix>:*)` pattern from the determined prefix
6. Sum up invocations that match the same pattern (e.g., `git status` and `git status -s` both count toward `Bash(git status:*)`)

#### Other Tools

| Tool | Pattern Generation Rule |
|------|----------------------|
| Read, Grep, Glob | Tool name only: `Read`, `Grep`, `Glob` |
| Edit | Detect the common parent directory of `input.file_path` values: `Edit(src/**)` |
| Write | Detect the common parent directory of `input.file_path` values: `Write(src/**)` |
| WebFetch | `WebFetch` |
| WebSearch | `WebSearch` |
| MCP tools (`mcp__*`) | Always per-tool: `mcp__github__create_pull_request` |
| Task, TaskList, TaskCreate, TaskUpdate, TaskGet, SendMessage, TeamCreate, TeamDelete, AskUserQuestion, Skill, NotebookEdit | **Excluded** (no permission configuration needed) |

#### Edit / Write Path Aggregation Rules

1. Convert all file paths to relative paths from the project root
2. Group by the top-level directory (first path segment)
3. If 2 or more files exist under the same directory, aggregate as `Edit(<dir>/**)`
4. If only 1 file, use the path as-is: `Edit(path/to/file.ts)`
5. If files span multiple top-level directories, create separate patterns for each (do not use bare `Edit`)

### Step 4: Read Existing Settings

Read the following 3 files with Read (treat as empty if they don't exist):

1. `~/.claude/settings.json` — user scope
2. `.claude/settings.json` — project scope (relative to current directory)
3. `.claude/settings.local.json` — local scope (relative to current directory)

Extract the `permissions.allow` and `permissions.deny` arrays from each file and merge across all scopes.

### Step 5: Filtering and Risk Classification

#### Exclusion Rules

1. Exclude patterns already covered by existing allow rules
   - Exact match: if `Read` already exists, exclude `Read`
   - Containment: if `Bash(git *:*)` already exists, exclude `Bash(git status:*)`
   - Containment: if `Read` already exists, exclude `Read(src/**)`
2. Exclude patterns that match existing deny rules
3. Display the count of deny-excluded items in the footer

#### Risk Classification

Assign a risk level to each candidate based on the following criteria:

**Low Risk (Recommended) — Read-only, no side effects:**
- `Read`, `Grep`, `Glob`, `WebSearch`
- `Bash(git status:*)`, `Bash(git diff:*)`, `Bash(git log:*)`, `Bash(git branch:*)`, `Bash(git show:*)`, `Bash(git stash list:*)`
- `Bash(ls:*)`, `Bash(cat:*)`, `Bash(wc:*)`, `Bash(head:*)`, `Bash(tail:*)`

**Medium Risk (Review) — File changes, local operations:**
- `Edit(...)`, `Write(...)`, `WebFetch`
- `Bash(git add:*)`, `Bash(git commit:*)`, `Bash(git stash:*)`, `Bash(git checkout:*)`, `Bash(git switch:*)`
- Test/build commands: `Bash(npm run test:*)`, `Bash(npm run lint:*)`, `Bash(pytest:*)`, `Bash(bundle exec rspec:*)`, `Bash(cargo test:*)`, `Bash(make test:*)`, etc.
- Package managers: `Bash(npm install:*)`, `Bash(pip install:*)`, `Bash(bundle install:*)`

**High Risk (Caution) — Remote impact, destructive operations:**
- `Bash(git push:*)`, `Bash(git pull:*)`, `Bash(git rebase:*)`, `Bash(git reset:*)`
- `Bash(rm:*)`, `Bash(mv:*)`, `Bash(chmod:*)`
- `Bash(docker compose down:*)`, `Bash(docker rm:*)`
- `Bash(npm publish:*)`, `Bash(cargo publish:*)`
- Write-oriented MCP tools

Default classification for patterns not listed above:
- Tool-name-only patterns (no arguments) → Medium Risk
- `Bash(...)` not matching any of the above → Medium Risk
- `mcp__*` patterns → Medium Risk

#### Sorting and Limits

- Within each risk level, sort by "session occurrence rate (how many sessions used it)" then by "invocation count", both descending
- Display up to 15 candidates maximum

### Step 6: Output Results

Output in the following Markdown format:

```
## grantly:suggest — Permission Candidates

Project: <project directory name>
Analyzed: last <N> sessions (<oldest date> ~ <newest date>)
Existing rules: allow <N> / deny <N> (<settings file paths, comma-separated>)

---

### Recommended (Low Risk) — Read-only, no side effects

  1. <pattern>                    <count> uses / <N> sessions: <occurrence>
  ...

### Review (Medium Risk) — File changes, local operations

  <N>. <pattern>                  <count> uses / <N> sessions: <occurrence>
  ...

### Caution (High Risk) — Remote impact, destructive operations

  <N>. <pattern>                  <count> uses / <N> sessions: <occurrence>
  ...

---

Select numbers to add (e.g., 1-4, 6)
Add all: a / Skip: s
```

- Omit a risk section if it has no candidates
- If candidates were excluded due to deny rules, append to the footer: `* <N> candidate(s) excluded by deny rules`
- If there are 0 candidates, display the following and stop:
  ```
  No new permission candidates found.
  Your current settings already cover the primary tool invocations.
  ```

### Step 7: Process User Selection

Wait for user input and process as follows:

1. **Parse number selection:**
   - `1,2,3` → individual selection
   - `1-4` → range selection
   - `1-4, 6, 8` → mixed
   - `a` → select all candidates
   - `s` → add nothing, display "No changes made." and stop

2. **Ask for target scope:**
   ```
   Select target scope:
     1. user   (~/.claude/settings.json)          — shared across all projects
     2. project (.claude/settings.json)           — shared with team (under VCS)
     3. local  (.claude/settings.local.json)      — this project only (gitignored)

   Selection [3]:
   ```
   - Default is `3` (local)
   - If `1` (user) is selected, confirm: "This will affect all projects. Are you sure?"

3. **Confirm the changes:**
   ```
   The following rules will be added to <file path>:

     allow:
       - <rule1>
       - <rule2>
       ...

   Apply? (y/n) [y]:
   ```

### Step 8: Write to Settings File

1. Read the target settings file with Read
2. If the file does not exist, initialize with `{}`
3. Add the `permissions` key if missing
4. Initialize `permissions.allow` as an empty array if missing
5. Append the selected rules to the `permissions.allow` array (skip duplicates)
6. Format JSON with 2-space indentation
7. Write the file with the Write tool
8. Display the result:
   ```
   Added <N> rule(s) to <file path>.
   Changes will take effect in your next session.
   ```

## Important Notes

- Session JSONLs contain user work details — never quote or display file contents or command arguments. Only output aggregated pattern statistics.
- Do not display intermediate progress (JSONL loading status, etc.). Only output the final result.
- If errors occur (JSONL parse errors, etc.), skip and continue.
