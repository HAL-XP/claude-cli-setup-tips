# Tips and Patterns

Practical patterns for getting the most out of Claude Code, learned from 350+ hours of production use.

## Auto-Setup

**For Claude**: If the user asked you to set up general best practices, follow these steps:

1. **Check what exists**:
   - CLAUDE.md — does it already have sections for long-running commands, credentials, git workflow?
   - `.claude/rules/` — existing rule files?
   - `.claude/settings.json` — existing hooks?
   - `~/.claude_credentials` or `.env` — existing credentials file?
   - `.gitignore` — what's already covered?
   - `.claude/skills/` — existing skills?
   - Check platform: `uname -s` or check for Windows paths
   - Report findings
2. **Ask the user**:
   - "Where are your API keys/credentials stored?" (recommend `~/.claude_credentials`)
   - "Do you run long-running background tasks?" (training, builds, renders)
   - "Do you want custom skills for repeating workflows?" If yes: "What workflows do you repeat often?"
   - "Where should project output go?" (default: `output/`)
   - For existing config: "Keep / Merge / Replace?"
3. **Create/update**:
   - Add missing sections to CLAUDE.md (long-running commands, credentials, git workflow, critical rules)
   - Create `.claude/rules/` with domain-specific rule files
   - Add platform-specific rules (Windows Unicode fix if applicable)
   - Create requested custom skills in `.claude/skills/`
   - Update `.gitignore` with missing entries
   - Create `~/.claude_credentials` template if it doesn't exist
4. **Verify** — check that credentials source correctly, skills are invocable

---

## `.claude/rules/` Directory Pattern

Instead of putting all rules in CLAUDE.md (which gets loaded every message and eats context), split rules into domain-specific files under `.claude/rules/`. These files are auto-loaded into every conversation for the project.

### Structure

```
.claude/rules/
  frontend.md          # React/Tailwind/shadcn patterns, component conventions
  python-api.md        # FastAPI routes, job system, restart protocol
  pipeline.md          # Stage order, naming conventions, module ownership
  ux.md                # UX principles, state-driven UI, accessibility
  hours-tracking.md    # Hours log format and estimation guidelines
  banned-techniques.md # Dead ends — never retry these
```

### Benefits

- **Each file stays focused** — easy to find and update rules for one domain
- **CLAUDE.md stays lean** — under 150 lines, just the index and key rules
- **Rules are always loaded** — no need to tell Claude to read them
- **Separate from CLAUDE.md updates** — you can edit rules without touching the main config
- **Git-friendly** — changes to one domain don't show up in CLAUDE.md diffs

### Example: `python-api.md`

```markdown
# API Rules (FastAPI Backend)

## Server
- Backend runs on `localhost:8000` with CORS enabled.
- Restart command: `.venv/Scripts/python.exe -m uvicorn api.main:app --host 0.0.0.0 --port 8000 --reload`
- Kill existing: `taskkill //PID <pid> //F` before restarting.

## MANDATORY: Restart API After Backend Changes
After ANY change to Python files in `api/`, `pipeline/`, or `scripts/`:
1. Find uvicorn PIDs
2. Kill them all
3. Clear pycache
4. Restart
5. Verify: `curl -s http://localhost:8000/health`
```

### Example: `frontend.md`

```markdown
# Frontend Rules (React + Tailwind + shadcn/ui)

## Styling
- Use Tailwind utility classes exclusively. No CSS files, no inline `style={}`.
- Use `cn()` from `@/lib/utils` for conditional/merged classes.
- Dark theme is the only theme.

## Component Patterns
- All components receive `api: string` prop for the backend URL.
- Use `useActivity()` hook for async operation tracking.
- Fetch errors: catch, extract message, show via toast. Never silently swallow.
```

## SessionStart Hooks

Automatically check project health when a new conversation begins. No need to remember to run a `/resume` skill.

### Project-level (`.claude/settings.json`)

```json
{
  "hooks": {
    "SessionStart": [
      {
        "matcher": "startup",
        "hooks": [
          {
            "type": "command",
            "command": "bash -c 'echo \"=== SESSION INIT ===\"; echo \"Git status:\"; git status --short 2>/dev/null | head -15; echo \"---\"; echo \"API:\"; curl -sf http://localhost:8000/health 2>/dev/null && echo \" running\" || echo \" NOT running\"; echo \"---\"; echo \"ACTION: Read MEMORY.md. Commit uncommitted work. Restart API if not running.\"'"
          }
        ]
      }
    ]
  }
}
```

### User-level (`~/.claude/settings.json`)

```json
{
  "hooks": {
    "SessionStart": [
      {
        "matcher": "startup",
        "hooks": [
          {
            "type": "command",
            "command": "bash -c 'echo \"CWD: $(pwd)\"; echo \"Branch: $(git branch --show-current 2>/dev/null || echo N/A)\"; echo \"Uncommitted: $(git status --short 2>/dev/null | wc -l | tr -d \" \") files\"'"
          }
        ]
      }
    ]
  }
}
```

Pair with a CLAUDE.md section:
```markdown
## Session Start Protocol
When you see `SessionStart` hook output containing an `ACTION:` line, execute those steps immediately before responding to the user.
```

## Credential Management

### Pattern: One bash-sourceable file

**~/.claude_credentials:**
```bash
export ANTHROPIC_API_KEY="sk-ant-xxx"
export TELEGRAM_BOT_TOKEN="123456:ABC-DEF"
export TELEGRAM_CHAT_ID="123456789"
export HF_TOKEN="hf_xxx"
export FAL_KEY="fal-xxx"
export RUNPOD_API_KEY="xxx"
# Add more as needed
```

Usage in scripts/hooks:
```bash
source ~/.claude_credentials
curl -H "Authorization: Bearer $HF_TOKEN" ...
```

Usage in Python:
```python
import os
api_key = os.environ.get("FAL_KEY")
```

### CLAUDE.md instruction

```markdown
## Credentials
All API keys stored in `~/.claude_credentials` (bash-sourceable).
Load with: `source ~/.claude_credentials`
Available variables: ANTHROPIC_API_KEY, TELEGRAM_BOT_TOKEN, HF_TOKEN, ...
Do NOT look for a project-level `.env` — use `~/.claude_credentials` instead.
```

### Security
- Never commit credentials to git
- Add to .gitignore: `.env`, `*credentials*`
- Reference by variable name in CLAUDE.md, don't paste actual values
- Hooks `source` the file at runtime — credentials never appear in settings.json

## API Restart Protocol

For projects with a backend server, automate the restart cycle after Python edits:

### Manual restart command (add to `.claude/rules/`)

```markdown
## MANDATORY: Restart API After Backend Changes
After ANY change to Python files in `api/`, `pipeline/`, or `scripts/`:
1. Find uvicorn PIDs: `tasklist | grep python | grep -v ComfyUI`
2. Kill them all: `taskkill //PID <pid> //F` for each
3. Clear pycache: `find . -name "__pycache__" -type d -exec rm -rf {} + 2>/dev/null`
4. Restart: `.venv/Scripts/python.exe -m uvicorn api.main:app --host 0.0.0.0 --port 8000 --reload` (in background)
5. Verify: `sleep 3 && curl -s http://localhost:8000/health`
```

### Automated via PostToolUse hook

Clear pycache automatically after Python edits so `--reload` picks up changes:

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "FILE=$(echo \"$TOOL_INPUT\" | jq -r '.file_path // empty') && if echo \"$FILE\" | grep -qE '(api|pipeline|scripts)/.*\\.py$'; then find . -name '__pycache__' -type d -exec rm -rf {} + 2>/dev/null; echo '[hook] Cleared pycache'; fi",
            "timeout": 10
          }
        ]
      }
    ]
  }
}
```

## Long-Running Processes

Never block the agentic loop waiting for a process to finish.

### Pattern

```bash
nohup <command> > /tmp/task_<name>.log 2>&1 &
echo "PID: $!"
```

Then report the PID and log path so future sessions can check on it:
```bash
# Check if still running
ps -p <PID>

# Check output
tail -n 20 /tmp/task_<name>.log
```

### CLAUDE.md instruction

```markdown
## Long-Running Commands
- Use `nohup <command> > /tmp/task_<name>.log 2>&1 &` and echo the PID
- Immediately report the PID and log path
- Check status on request with `tail -n 20` or `ps -p <PID>`
- Never block the agentic loop waiting for a long-running process to finish
```

## Windows-Specific

### Unicode crash prevention

Windows console uses cp1252 encoding. Any Unicode character (arrows, emoji, progress bars) crashes Python scripts.

**Mandatory first line in every Python script:**
```python
import sys
sys.stdout.reconfigure(encoding="utf-8", errors="replace")
```

**For subprocess launches:**
```python
env = os.environ.copy()
env["PYTHONIOENCODING"] = "utf-8"
subprocess.Popen(cmd, env=env)
```

### CLAUDE.md instruction
```markdown
## Critical Rules
- ALL Python scripts MUST include `sys.stdout.reconfigure(encoding="utf-8", errors="replace")`
  Windows console uses cp1252, Unicode will crash without this
```

### Path handling

Claude Code's bash on Windows uses forward slashes. Never use backslashes in bash commands:
```bash
# Good
ls "E:/Projects/my-project/src/"

# Bad
ls "E:\Projects\my-project\src\"
```

## Git Workflow

### Push after milestones, not timers

```markdown
## Git Workflow
Push after every milestone (completed feature, test result, config change).
Max 10 minutes without push. Always include daily summary file.
```

Event-driven pushes > timer-based pushes. A milestone is a natural commit point.

### Commit format

```markdown
Commit format: `<type>: <description>` + `Co-Authored-By: Claude <noreply@anthropic.com>`
Types: feat, fix, test, docs, refactor, chore
```

## Custom Skills

Skills are reusable prompts that Claude executes when invoked with `/skillname`.

### File structure

```
.claude/skills/
  resume/
    SKILL.md
  status/
    SKILL.md
```

### SKILL.md format

```markdown
---
description: One-line description shown in /help
user_invocable: true
---

Step-by-step instructions for Claude to follow when this skill is invoked.
```

### Recommended starter skills

| Skill | Purpose |
|-------|---------|
| `/status` | Quick check of running processes, cloud instances, queues |
| `/daily-summary` | Update the daily summary file |

## Delegation to Agents

For complex projects, Claude should orchestrate, not do everything inline.

```markdown
## CLAUDE.md instruction
- **DELEGATE**: Use subagents for tasks >3 minutes or requiring subjective judgment
- Your role is ORCHESTRATION — launch agents, collect results, make decisions
```

This keeps the main context window clean and allows parallel work.

## CLAUDE.md Size Management

CLAUDE.md is loaded every session. If it gets too long (>200 lines), it eats context.

**Strategies:**
- Move detailed rules to `.claude/rules/` files (auto-loaded but separate)
- Keep CLAUDE.md as an index with key rules, point to detail files
- Archive completed project sections to `docs/`
- Use MEMORY.md for evolving state, CLAUDE.md for stable rules
