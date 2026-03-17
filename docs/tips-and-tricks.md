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

Chat gets compacted. Strategy analysis, research findings, experiment results -- if they only exist in the conversation, they are gone after compaction. Save them:

```
output/strategy/analysis_20260310.md
output/decisions/2026-03-12_auth-architecture.md
```

## Long-Running Processes

Never block the agentic loop waiting for a process to finish.

```bash
nohup <command> > /tmp/task_<name>.log 2>&1 &
echo "PID: $!"
```

Check on it later:

```bash
ps -p <PID>
tail -n 20 /tmp/task_<name>.log
```

CLAUDE.md instruction:

```markdown
## Long-Running Commands
- Use `nohup <command> > /tmp/task_<name>.log 2>&1 &` and echo the PID
- Never block the agentic loop waiting for a process to finish
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
