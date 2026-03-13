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
10. Do I want automatic health checks on session start? (SessionStart hooks)
11. Do I want automatic type-checking/linting after edits? (PostToolUse hooks)
12. Do I need cloud GPU support? (Vast.ai, RunPod, Lambda, etc.)

---

Based on my answers, create ALL of the following:

## 1. CLAUDE.md (project root)

The main instruction file Claude reads every session. Keep it under 150 lines. Include:

**Project Goal section:**
- Project name, goal, active workstreams
- Current status and key metrics

**Session Continuity section (CRITICAL):**
- On session start: check MEMORY.md, check SessionStart hook output, check running PIDs
- On session end: update MEMORY.md with current task, next action, any running processes
- This prevents context loss between sessions

**Session Start Protocol:**
- When SessionStart hook output contains an ACTION: line, execute those steps immediately
- Don't wait for the user to ask. Don't ask permission. Just do it.

**Operational Notes:**
- My timezone (ask me)
- Where credentials are stored (`~/.claude_credentials`, bash-sourceable)
- How to run long tasks without blocking: `nohup <cmd> > /tmp/task_<name>.log 2>&1 &`
- Push frequency (recommend: after every milestone)

**Critical Rules:**
- Project-specific invariants and gotchas

**Key Files table:**
| File | Purpose |
|------|---------|
(populated based on project)

**Git Workflow:**
- What to track vs. ignore
- Commit format: `<type>: <description>` (types: feat, fix, test, docs, refactor, chore)
- Include `Co-Authored-By: Claude <noreply@anthropic.com>` in commits

## 2. .claude/rules/ folder

Create `.claude/rules/` for domain-specific rule files. Files here are auto-loaded into every conversation. Keep CLAUDE.md as a lean index, put detailed rules in separate files.

Recommended structure:
- `banned-techniques.md` — Track approaches that don't work (empty template to start)
- One file per domain (e.g., `frontend.md`, `api.md`, `pipeline.md`) based on project structure
- Each file focused on one domain's conventions, patterns, and gotchas

## 3. .claude/settings.json (project-level hooks)

Create project-level hooks for automatic health checks:

**SessionStart hook** — runs every conversation start:
- Check git status
- Check if backend server is running (if applicable)
- Output ACTION: line telling Claude what to do

**PostToolUse hooks** (if applicable):
- TypeScript type-check after .tsx/.ts edits
- Pycache clear after Python edits (for uvicorn --reload)
- ESLint after JS edits

## 4. Auto-memory

Claude Code has built-in auto-memory at `~/.claude/projects/<project-hash>/memory/`. Seed it with an initial MEMORY.md:
```markdown
# Project Memory

## Current State
- (nothing yet — update this every session end)

## Key Technical Findings
- (populated as we discover things)

## Architecture Decisions
- (record important choices and WHY)

## User Preferences
- (workflow preferences, tool choices, communication style)
```

Explain that this file persists across conversations and Claude reads it at the start of every session.

## 5. .gitignore

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

## 6. User-level hooks (~/.claude/settings.json)

If I want notifications, help me configure user-level hooks:

**PreCompact hook** (fires before context window compaction):
- Echo instructions to update MEMORY.md before context is lost
- Send notification if configured
- This is the MOST important hook — prevents losing work-in-progress

**Permission prompt hook** (fires when Claude needs approval):
- Send notification so I know to check the terminal

**Idle notification hook** (smart relay):
- Claude writes to `/tmp/claude_telegram_msg.txt` before going idle
- Hook reads file, sends via Telegram, deletes file
- No file = no notification = no spam

Supported notification methods:
- **Telegram**: Create a bot via @BotFather, get chat_id via getUpdates API
- **Email**: Via sendmail or API
- **Desktop**: Via OS notification command
- **None**: Skip notification hooks, keep PreCompact echo-only

## 7. Skills (if repeating workflows exist)

If the project has repeating multi-step workflows, create `.claude/skills/` with:
- Each skill in its own subfolder with a SKILL.md
- At minimum suggest `/status` (check running things)

## 8. Platform-specific notes

**Windows:**
- Every Python script needs: `sys.stdout.reconfigure(encoding="utf-8", errors="replace")` at the top (console encoding issue)
- Use forward slashes in bash commands, not backslashes

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
| `.claude/rules/*.md` | Domain-specific rule files (auto-loaded) |
| `.claude/settings.json` | Project-level hooks (SessionStart, PostToolUse) |
| `~/.claude/settings.json` hooks | User-level hooks (PreCompact, Notification) |
| Auto-memory `MEMORY.md` | Persistent cross-session state |
| `.gitignore` | Keep secrets and temp files out of git |
| `.claude/skills/` | Reusable multi-step workflows |

## Key principles behind this setup

1. **Session continuity** — Auto-memory bridges the gap between conversations. SessionStart hooks automatically check project health. PreCompact hooks protect against context loss.
2. **Two levels of hooks** — User-level (`~/.claude/settings.json`) for global behavior (notifications, PreCompact). Project-level (`.claude/settings.json`) for project-specific automation (health checks, type-checking).
3. **Rules in `.claude/rules/`** — Keep CLAUDE.md lean (under 150 lines). Detailed domain rules go in separate files that are auto-loaded.
4. **Credentials centralization** — One file (`~/.claude_credentials`), sourced by bash. Never in project dirs, never in git.
5. **Banned techniques** — Track what doesn't work. Without this, you'll retry failed approaches across sessions.
6. **File-first outputs** — Anything important goes to a file, not just chat. Chat gets compacted; files persist.
