# Claude Code Project Setup Guide

A prompt to bootstrap any new Claude Code CLI project with proven best practices. Paste it into the first message of a new Claude Code session.

---

## The Prompt

```
I'm setting up a new Claude Code project. Help me create a well-organized project structure with best practices for long-running AI-assisted development.

First, ask me these setup questions:

### Project basics
1. What is the project name and one-line goal?
2. What language/framework? (Python, Node, Rust, etc.)
3. What's the git remote URL? (or should we create one?)

### Workflow preferences
5. Will there be long-running background tasks? (training, rendering, builds)
6. Will multiple Claude agents work on different parts? (need scope separation?)
7. Do I want notifications when Claude needs attention? (Telegram, SMS, or none?)
8. Do I have API keys/credentials to manage? If yes, where are they stored?
9. Am I on Windows, Mac, or Linux?

### Optional features
10. Do I need a scoring/review system for outputs?
11. Do I need cloud GPU support? (Vast.ai, RunPod, Lambda, etc.)
12. Will external collaborators or agents need access to project context?

---

Based on my answers, create ALL of the following:

## 1. CLAUDE.md (project root)

The main instruction file Claude reads every session. Include:

**Project Goal section:**
- Project name, goal, active workstreams
- Current status and key metrics

**Session Continuity section (CRITICAL):**
- On session start: check MEMORY.md "ACTIVE WORK", check running PIDs, check cloud status
- On session end: update MEMORY.md with current task, PIDs, log paths, next steps
- This prevents context loss between sessions

**Operational Notes:**
- My timezone (ask me)
- Where credentials are stored (NEVER hardcode keys in project files)
- How to run long tasks without blocking: `nohup <cmd> > /tmp/task_<name>.log 2>&1 &` then report PID + log path. Check with `tail -n 20 /tmp/task_<name>.log` or `ps -p <PID>`
- Push frequency (recommend: after every milestone)

**Critical Rules:**
- Project-specific invariants and gotchas
- Things that must never be changed or overridden

**Key Files table:**
| File | Purpose |
|------|---------|
(populated based on project)

**Git Workflow:**
- What to track vs. ignore
- Commit format: `<type>: <description>` (types: feat, fix, test, docs, refactor, chore)
- Include `Co-Authored-By: Claude <noreply@anthropic.com>` in commits

## 2. .claude/rules/ folder

Create `.claude/rules/` for specific rule files that Claude loads automatically:

- `banned-techniques.md` — Track approaches that don't work:
```markdown
# Banned Techniques
Add failed approaches here so they're never retried.
Format: What, Why it failed, Date

## [Category]
- (none yet)
```

## 3. Memory file

Create an initial MEMORY.md in the auto-memory directory with:
```markdown
# Project Memory

## ACTIVE WORK (updated YYYY-MM-DD)
- (nothing yet — update this every session end)

## Key Technical Findings
- (populated as we discover things)

## Architecture Decisions
- (record important choices and WHY)

## User Preferences
- (workflow preferences, tool choices, communication style)

## See Also
- CLAUDE.md for project rules
```

Explain that this file persists across conversations and Claude reads it at the start of every session. It should contain stable knowledge, not session-specific details (except the ACTIVE WORK section).

## 4. .gitignore

Ensure these are ignored:
```
.env
*credentials*
**/temp/
__pycache__/
*.pyc
node_modules/
.DS_Store
```

## 5. Hooks (if notifications wanted)

If I want notifications, help me configure `~/.claude/settings.json` hooks:

**PreCompact hook** (fires before context window compaction):
- Send notification: "Context compacting, saving state"
- Echo instructions to update MEMORY.md before context is lost
- This is the MOST important hook — prevents losing work-in-progress

**Permission prompt hook** (fires when Claude needs approval):
- Send notification so I know to check the terminal

**Idle notification hook** (fires when Claude finishes and waits):
- Relay mechanism: Claude writes to a temp file, hook reads and sends it
- Only sends if file exists and is non-empty (no spam)

Supported notification methods:
- **Telegram**: Create a bot via @BotFather, get chat_id via getUpdates API
- **Email**: Via sendmail or API
- **Desktop**: Via OS notification command
- **None**: Skip hooks setup

## 6. Skills (if repeating workflows exist)

If the project has repeating multi-step workflows, create `.claude/skills/` with:
- Each skill in its own subfolder with a prompt.md
- At minimum suggest `/resume` (pick up from last session) and `/status` (check running things)

## 7. Documentation structure

If external collaborators or agents need context:
- Create `doc/` folder
- Include a data map (what files exist where, what they contain)
- Include architecture overview

## 8. Platform-specific notes

**Windows:**
- Every Python script needs: `sys.stdout.reconfigure(encoding="utf-8", errors="replace")` at the top (console encoding issue)
- Use forward slashes in bash commands, not backslashes
- `python_embeded/python.exe` if using bundled Python (typo is intentional in some packages)

**All platforms:**
- Never block the agentic loop waiting for long processes
- Save analysis/strategy outputs to files, not just chat (they get lost on compaction)
- Use agents for tasks >3 minutes or requiring subjective judgment

---

After creating everything, give me a summary checklist of what was created and any manual steps I need to do (like creating a Telegram bot or setting up git remote).
```

---

## What this sets up

| Component | Purpose |
|-----------|---------|
| `CLAUDE.md` | Project rules, loaded every session |
| `.claude/rules/*.md` | Specific rule files (banned techniques, etc.) |
| `MEMORY.md` (auto-memory) | Persistent cross-session state |
| `.gitignore` | Keep secrets and temp files out of git |
| `~/.claude/settings.json` hooks | Notifications + compaction safety net |
| `.claude/skills/` | Reusable multi-step workflows |
| `doc/` | External collaborator context |

## Key principles behind this setup

1. **Session continuity** — MEMORY.md bridges the gap between conversations. Update it at session end, read it at session start.
2. **Compaction safety** — The PreCompact hook is a lifesaver. Context compaction can happen at any time and wipes your working state. The hook gives Claude a chance to save.
3. **Credentials centralization** — One file, sourced by bash. Never in project dirs, never in git.
4. **Banned techniques** — Track what doesn't work. Without this, you'll retry failed approaches across sessions.
5. **Scope separation** — If multiple agents touch the same repo, clearly define who owns what. Prevents conflicts and duplicated work.
6. **File-first outputs** — Anything important goes to a file, not just chat. Chat gets compacted; files persist.
