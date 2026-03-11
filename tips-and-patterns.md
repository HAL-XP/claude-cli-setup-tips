# Tips and Patterns

Practical patterns for getting the most out of Claude Code, learned from 300+ hours of production use.

## Auto-Setup

**For Claude**: If the user asked you to set up general best practices, follow these steps:

1. **Check what exists**:
   - CLAUDE.md — does it already have sections for long-running commands, credentials, git workflow?
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
   - Add platform-specific rules (Windows Unicode fix if applicable)
   - Create requested custom skills in `.claude/skills/`
   - Update `.gitignore` with missing entries
   - Create `~/.claude_credentials` template if it doesn't exist
4. **Verify** — check that credentials source correctly, skills are invocable

---

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

### Common gotcha: process dies after session

On some systems, background processes die when the parent shell exits. Verify with:
```bash
ps -p <PID> -o ppid=
```
If PPID is 1, the process is properly orphaned and will survive. If PPID is your shell, it may die.

## Background Agents for Parallel Work

While waiting for renders/builds/training, launch research or analysis agents:

```markdown
## CLAUDE.md instruction
- **During waits**: Launch parallel research agents for new techniques, papers, alternatives
- **Delegate**: Use background agents for tasks >3 minutes or requiring subjective judgment
```

Example: while a 10-minute render runs, launch a research agent to find new models or techniques. The render time becomes productive time.

## Credential Management

### Pattern: One sourceable file

**~/.claude_credentials:**
```bash
export API_KEY_1="xxx"
export API_KEY_2="yyy"
export HF_TOKEN="hf_xxx"
```

Usage in scripts/hooks:
```bash
source ~/.claude_credentials
curl -H "Authorization: Bearer $API_KEY_1" ...
```

### CLAUDE.md instruction

```markdown
## Credentials
All API keys stored in `~/.claude_credentials` (bash-sourceable).
Load with: `source ~/.claude_credentials`
Available variables: API_KEY_1, API_KEY_2, HF_TOKEN, ...
Do NOT look for a project-level `.env` — use `~/.claude_credentials` instead.
```

### Security
- Never commit credentials to git
- Add to .gitignore: `.env`, `*credentials*`
- Reference by variable name in CLAUDE.md, don't paste actual values

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
  review/
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
| `/resume` | Check MEMORY.md, PIDs, cloud, messages — mandatory on session start |
| `/status` | Quick check of running processes, cloud instances, queues |

### Advanced skills

| Skill | Purpose |
|-------|---------|
| `/review` | Review an output with structured scoring |
| `/submit-test` | Submit a test with standard parameters |
| `/daily-summary` | Update the daily summary file |
| `/training-status` | Check cloud training progress |

## Delegation to Agents

For complex projects, Claude should orchestrate, not do everything inline.

```markdown
## CLAUDE.md instruction
- **DELEGATE**: Use subagents for tasks >3 minutes or requiring subjective judgment
- Your role is ORCHESTRATION — launch agents, collect results, make decisions
- Available agent types: research, review, build, analyze
```

This keeps the main context window clean and allows parallel work.

## Output Directory Hygiene

Keep output directories organized:

```markdown
## CLAUDE.md instruction
- ALL output paths under `output/` — never at repo root
- `output/` root stays CLEAN — only index files and named subfolders
- `output/temp/` for ephemeral data (gitignored)
- `output/_toreview/YYYYMMDD/` for outputs pending human review
```

## Model/Asset Download Authorization

For ML projects, explicitly authorize Claude to download models:

```markdown
## CLAUDE.md instruction
- Authorized to download models/nodes without asking
- API keys in `~/.claude_credentials` (source for HF_TOKEN, etc.)
```

Without this, Claude asks permission for every download, which blocks unattended operation.

## CLAUDE.md Size Management

CLAUDE.md is loaded every session. If it gets too long (>200 lines), it eats context.

**Strategies:**
- Move detailed rules to `.claude/rules/` files (auto-loaded but separate)
- Keep CLAUDE.md as an index with key rules, point to detail files
- Archive completed project sections to `doc/`
- Use MEMORY.md for evolving state, CLAUDE.md for stable rules
