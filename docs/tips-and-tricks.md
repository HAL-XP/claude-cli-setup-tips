# Tips, Tricks & Lessons Learned

Hard-won lessons from 500+ hours of production Claude Code development.

## Rules of Thumb

### When to Spawn an Agent

**If the task requires more than 1 Edit/Write tool call, spawn an agent.**

This keeps the orchestrator's context clean for decision-making. Even "quick fixes" tend to snowball.

### Never Trust Status Badges

Agent says "TypeScript compiles clean" and "looks correct"? That is UNVERIFIED, not PASS. The only way to verify is to RUN it:
- `curl` the endpoint
- `npx tsc --noEmit` for TypeScript
- Playwright screenshot for UI
- `pytest` for tests

Code review is never functional verification.

### Think Systemically, Not Per-Instance

Never fix a single instance of a problem. Ask "why did this happen?" and fix the root cause:
- If one asset is misaligned, fix the detection/correction system
- If one popup can't be clicked, fix the popup framework
- If one API endpoint crashes, fix the error handling pattern

Every bug is a signal that a system is missing or broken. Build the system.

### Design for Scale

- A project may have 5 items or 500
- Every UI must handle 0, 1, few, and many items
- Batch operations must support partial failure and progress tracking
- Search/filter/sort must exist anywhere a list can grow beyond 10 items

### Autonomous Verification

After implementing a feature, verify it works yourself:
- Use Playwright MCP to screenshot and inspect UI changes
- Hit API endpoints with curl to verify responses
- Run the actual pipeline stage on test data
- If you cannot verify, say so explicitly -- do not assume it works

### File-First Outputs

**Anything important goes to a file, not just chat.**

Chat gets compacted. Strategy analysis, research findings, experiment results -- if they only exist in the conversation, they are gone after compaction. Save them to the `_devlog/` directory:

```
_devlog/
  summaries/summary_20260310.md       # Session work log
  hours/hours_20260310.md             # Human-equivalent hours
  decisions/20260312_auth-arch.md     # Architecture decision records
  experiments/20260310_lpips.md       # Trial results (adopt or kill)
```

See [Memory > Devlog](memory.md#devlog-_devlog-optional) for the full structure and templates.

## CLI Responsiveness (Never Block the Terminal)

**The #1 quality-of-life rule:** Never run a foreground command that takes more than ~5 seconds. The user loses the ability to type, interrupt (Ctrl+B), or redirect — extremely frustrating.

### Use `run_in_background: true`

For any Bash tool call expected to take more than a few seconds:

```json
{
  "command": "python train.py --epochs 100",
  "run_in_background": true
}
```

Then check on it with `TaskOutput`:

```json
{
  "task_id": "abc123",
  "block": false,
  "timeout": 5000
}
```

### What must run in background

- Generation scripts (TTS, image, model inference)
- Downloads (models, datasets, large files)
- Installs (`pip install`, `npm install`, `choco install`)
- Audio/video playback
- Builds and compilations
- Anything with a loop, poll, or timeout
- Test suites

### CLAUDE.md instruction

```markdown
## CLI Responsiveness
Never block the terminal. Any command expected to take more than ~5 seconds
MUST use `run_in_background: true`. Keep the CLI interactive at all times.
Use `TaskOutput` to check on background tasks when needed.
```

## Long-Running Processes (Detached)

For processes that must survive session closure (servers, training jobs), use `nohup`:

```bash
nohup <command> > /tmp/task_<name>.log 2>&1 &
echo "PID: $!"
```

Check on it later:

```bash
ps -p <PID>
tail -n 20 /tmp/task_<name>.log
```

Note: `run_in_background` is for commands within the session. `nohup` is for processes that must outlive the session.

## Process Management (PID Tracking)

**NEVER kill processes by name.** `taskkill //IM electron.exe //F` or `pkill python` carpet-bombs every instance on the machine — including other projects.

**ALWAYS kill by PID**, using the process tree flag to catch child processes:

```bash
# Windows
taskkill //PID <pid> //T //F

# macOS / Linux
kill -TERM <pid>
```

### Track PIDs in `.claude/.pids`

When launching any long-running process, save its PID to a project-local file:

```json
{
  "dev-server": 12345,
  "api": 67890,
  "worker": 11111
}
```

**Launching:**

```bash
# Save PID when launching a background process
nohup npm run dev > /tmp/dev-server.log 2>&1 &
PID=$!
# Write to .claude/.pids (create if needed)
node -e "
  const fs = require('fs');
  const f = '.claude/.pids';
  const pids = fs.existsSync(f) ? JSON.parse(fs.readFileSync(f)) : {};
  pids['dev-server'] = $PID;
  fs.writeFileSync(f, JSON.stringify(pids, null, 2));
"
```

**Killing:**

```bash
# Read PID and kill only that process tree
PID=$(node -e "const p=JSON.parse(require('fs').readFileSync('.claude/.pids'));console.log(p['dev-server']||'')")
if [ -n "$PID" ]; then
  taskkill //PID $PID //T //F 2>/dev/null   # Windows
  # kill -TERM $PID 2>/dev/null             # macOS/Linux
fi
```

**Self-registering from an app:**

Electron, Node, or Python apps can write their own PID on startup:

```typescript
// In Electron main process or Node server entry
import { writeFileSync, readFileSync, mkdirSync, existsSync } from 'fs'

const PIDS_FILE = '.claude/.pids'
function registerPid(name: string) {
  mkdirSync('.claude', { recursive: true })
  const pids = existsSync(PIDS_FILE) ? JSON.parse(readFileSync(PIDS_FILE, 'utf-8')) : {}
  pids[name] = process.pid
  writeFileSync(PIDS_FILE, JSON.stringify(pids, null, 2))
}

function unregisterPid(name: string) {
  try {
    const pids = JSON.parse(readFileSync(PIDS_FILE, 'utf-8'))
    delete pids[name]
    writeFileSync(PIDS_FILE, JSON.stringify(pids, null, 2))
  } catch {}
}

registerPid('electron')
process.on('exit', () => unregisterPid('electron'))
process.on('SIGTERM', () => { unregisterPid('electron'); process.exit(0) })
```

**Add to .gitignore:**

```gitignore
.claude/.pids
```

### CLAUDE.md / Rules Instruction

Add to your project's rules or CLAUDE.md:

```markdown
## Process Management
- NEVER kill by process name — always by PID from `.claude/.pids`
- Save PIDs when launching background processes
- Kill with `taskkill //PID <pid> //T //F` (Windows) or `kill -TERM <pid>` (macOS/Linux)
- Verify the PID still exists before killing: `tasklist //FI "PID eq <pid>"` (Windows)
```

## API Restart Protocol

For projects with a backend server, automate the restart cycle:

```markdown
## MANDATORY: Restart API After Backend Changes

After ANY change to Python files:
1. Find PIDs: `tasklist | grep python`
2. Kill: `taskkill //PID <pid> //F`
3. Clear pycache: `find . -name "__pycache__" -type d -exec rm -rf {} +`
4. Restart in background
5. Verify: `sleep 3 && curl -s http://localhost:8000/health`

Do this AFTER committing, as the final step. Never skip it.
```

Automate pycache clearing with a PostToolUse hook (see [Hooks](hooks.md)).

## Git Workflow

### Push After Milestones, Not Timers

Event-driven pushes are better than timer-based. A milestone is a natural commit point:
- Feature complete
- Test passing
- Bug fixed
- Configuration change

### Commit Format

```
<type>: <description>

Co-Authored-By: Claude <noreply@anthropic.com>
```

Types: `feat`, `fix`, `test`, `docs`, `refactor`, `chore`

### Never Skip Hooks

Do not use `--no-verify` or `--no-gpg-sign` unless the user explicitly asks. If a pre-commit hook fails, fix the issue and create a NEW commit (do not amend).

## Custom Skills

Skills are reusable prompts that Claude executes when invoked with `/skillname`.

```
.claude/skills/
  status/
    SKILL.md
  daily-summary/
    SKILL.md
```

SKILL.md format:

```markdown
---
description: Quick check of running processes and project health
user_invocable: true
---

Check the following and report status:
1. Git status and recent commits
2. API server health (curl localhost:8000/health)
3. Any running background processes
4. MEMORY.md current state
```

## Key Slash Commands

| Command | Purpose |
|---------|---------|
| `/init` | Generate starter CLAUDE.md |
| `/memory` | View/edit memory files and rules |
| `/agents` | Manage agent templates |
| `/hooks` | View configured hooks |
| `/compact` | Manually compact context |
| `/effort` | Set model effort level |
| `/voice` | Voice mode (push-to-talk) |
| `/loop 5m <prompt>` | Run prompt on recurring interval |
| `/batch <prompt>` | Parallel codebase migration |
| `/btw <question>` | Side question (no context impact) |
| `/simplify` | Review changed code for reuse, quality, efficiency |
| `/color` | Customize UI colors |
| `/copy` | Copy last response (optional index: `/copy N`) |
| `/plan <desc>` | Enter plan mode with optional description |
| `/context` | Show loaded context with actionable suggestions |
| `/plugin install` | Install a plugin from a marketplace |
| `/remote-control` | Bridge sessions to claude.ai/code (VS Code) |

### /simplify (Code Quality Review)

The `/simplify` bundled skill reviews your changed code for reuse opportunities, quality issues, and efficiency improvements. Run it after finishing a feature or refactor:

```
/simplify
```

Combine with `/loop` for recurring quality checks:

```
/loop 30m /simplify
```

## Engineering Anti-Patterns

### The "Quick Fix" Trap

Orchestrator thinks "I'll just fix this one thing" and edits a file inline. Then another. Then another. Soon the orchestrator's context is full of implementation details and it loses the big picture.

**Rule**: The orchestrator's job is to THINK, not to CODE.

### The "Obviously Works" Fallacy

"The change is so simple it obviously works." This is how bugs ship. Even "trivial" changes need verification:
- A one-line CSS change can break responsive layout
- A renamed field can break 5 API consumers
- A "harmless" refactor can change execution order

### The "tsc Clean = Done" Trap

TypeScript compiling without errors means the TYPES are correct. It does not mean:
- The feature renders correctly
- The API returns the right data
- The user flow works end-to-end
- The empty/error/loading states are handled

### The "I'll Remember" Trap

You will not remember across sessions. Write it down:
- MEMORY.md for current state
- Banned techniques for dead ends
- Decision documents for architectural choices
- File-first for all analysis

## Platform-Specific Tips

### Windows

- `sys.stdout.reconfigure(encoding="utf-8", errors="replace")` in every Python script
- Forward slashes in all bash paths
- `taskkill //PID <pid> //F` (double slashes)
- Claude Code's bash is Git Bash -- use Unix syntax, not Windows

### macOS

- `jq` may need to be installed separately (`brew install jq`)
- tmux works well for split-pane agent teams

### Large Monorepos

- Use `claudeMdExcludes` to skip irrelevant CLAUDE.md files
- Use `worktree.sparsePaths` for faster worktree creation
- Use path-scoped rules to limit context noise

## Settings You Should Know

```json
{
  "effortLevel": "high",
  "skipDangerousModePermissionPrompt": true,
  "enableRemoteControlForAllSessions": true,
  "autoUpdatesChannel": "latest"
}
```

| Setting | What it does |
|---------|-------------|
| `effortLevel` | `low`, `medium`, `high` -- controls model reasoning depth |
| `skipDangerousModePermissionPrompt` | Skip the "are you sure?" when using bypass mode |
| `enableRemoteControlForAllSessions` | Allow remote control of sessions |
| `autoUpdatesChannel` | `latest` or `stable` for auto-updates |
| `autoMemoryDirectory` | Custom path for auto-memory files (default: `~/.claude/projects/`) |

### --bare Flag (Scripted Automation)

When using `claude -p` in scripts or CI, the `--bare` flag skips hooks, LSP, and plugin sync for faster execution:

```bash
claude --bare -p "What is the main entry point of this project?"
```

Use this for scripted queries where you don't need the full session setup. Added in v2.1.81.

### 1M Context Window

Opus 4.6 with Max/Team/Enterprise plans now supports 1M token context (since v2.1.75). Practical implications:

- **Less frequent compaction**: You can work longer before context is compressed
- **Larger files readable**: Can read very large files in a single pass
- **More agent context**: Subagents inherit the larger window
- **Memory strategy change**: With 1M context, MEMORY.md is less critical for short sessions but still essential for cross-session persistence

This doesn't change the fundamentals — still use PreCompact hooks, still keep MEMORY.md updated, still use file-first outputs. But you'll hit compaction less often.

## Session Name

Use the `-n` flag to name your session:

```bash
claude -n "frontend-refactor"
```

This helps identify sessions in logs and notifications.

## Status Line

Configure a custom status line that shows in the Claude Code UI:

```json
{
  "statusLine": {
    "type": "command",
    "command": "bash /path/to/statusline.sh"
  }
}
```

Configure with `/statusline`.
