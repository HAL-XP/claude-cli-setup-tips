# Claude Code Project Setup Kit

Battle-tested patterns for getting the most out of Claude Code CLI, learned from 350+ hours of production development across multi-agent workflows.

Framework-agnostic — works for any project where Claude Code is your development partner.

## Important: Claude Code Has Built-In Memory Now

Claude Code now includes **auto-memory** out of the box. It stores persistent memories at `~/.claude/projects/<project-hash>/memory/` and loads them into every conversation. You do not need to build a custom memory system from scratch.

This kit focuses on the patterns and workflows that go **beyond** what auto-memory provides: hooks, notifications, multi-agent coordination, research tracking, and project organization.

## Installation

Copy the guide files into your project. Recommended location: `docs/claude-setup/`

```bash
# Clone this repo (or download as ZIP)
git clone https://github.com/YOUR_USER/claude-cli-setup-tips.git /tmp/claude-setup

# Copy into your project
cp -r /tmp/claude-setup/*.md your-project/docs/claude-setup/
```

Or just copy the files wherever makes sense for your project — the guides reference each other by filename (relative), so any single folder works.

## Quick Start

Once the files are in your project, open Claude Code in your project directory and say:

```
Read docs/claude-setup/SETUP.md and set up this project
```

Adjust the path to wherever you put the files. Claude asks 3-4 essential questions, creates the core files (CLAUDE.md, `.claude/rules/`, hooks, etc.), then shows optional features you can add anytime.

**Re-runnable**: Come back weeks later and say "set up research tracking" or "add notifications" — Claude detects what exists and only adds what's missing.

**Individual guides**: Each guide works standalone too:
```
Read docs/claude-setup/notification-hooks.md and set it up for this project
```

## What's Inside

### Core (set up first)

| File | What it covers |
|------|---------------|
| **[SETUP.md](SETUP.md)** | Master installer — detects existing config, asks only what's needed, creates files, shows available add-ons. Start here. |
| **[session-continuity.md](session-continuity.md)** | Auto-memory system, MEMORY.md, pre-compact hooks, daily summaries, file-first output rule. Ensures no work is lost between sessions. |

### Add-ons (enable when you need them)

| File | What it covers | Best for |
|------|---------------|----------|
| **[notification-hooks.md](notification-hooks.md)** | SessionStart, PostToolUse, PreCompact, Notification hooks. Telegram/SMS/desktop. Smart relay pattern. Both user-level and project-level examples. | Remote work, AFK monitoring, automation |
| **[multi-agent-coordination.md](multi-agent-coordination.md)** | Filesystem message board for separate CLI agent instances, shared context directory, scope separation, `[AgentName]` prefix convention. | 2+ Claude Code sessions on same project |
| **[research-tracking.md](research-tracking.md)** | Banned techniques (auto-loaded dead ends), LEARNINGS.json, decision docs, progress trees, research registries. | Any iterative project — dev, ML, research |
| **[hours-tracking.md](hours-tracking.md)** | End-of-session time tracking — maps tasks to human-equivalent hours with multiplier. | ROI analysis, project planning |
| **[tips-and-patterns.md](tips-and-patterns.md)** | `.claude/rules/` directory pattern, SessionStart hooks, credentials, API restart protocol, Windows Unicode fixes, git workflow, custom skills, CLAUDE.md management. | Everyone — grab bag of essentials |

### Reference

| File | What it is |
|------|-----------|
| **[project-setup-prompt-generic.md](project-setup-prompt-generic.md)** | Alternative to SETUP.md — a self-contained prompt to paste into a new Claude Code session. Doesn't require copying files into your project. |

## What These Patterns Solve

| Problem | Pattern | Guide |
|---------|---------|-------|
| "Where was I?" after restarting | Auto-memory + MEMORY.md | session-continuity |
| Context compaction destroys work | PreCompact hook saves state | session-continuity |
| Retrying approaches that already failed | Banned techniques file (auto-loaded) | research-tracking |
| Copy-pasting between two Claude terminals | Filesystem message board | multi-agent-coordination |
| Claude blocked on permission, user is AFK | Permission prompt notification | notification-hooks |
| Important analysis lost after compaction | File-first output rule | session-continuity |
| No idea how much time Claude is saving | Hours tracking at session end | hours-tracking |
| Python crashes on Windows with Unicode | `sys.stdout.reconfigure` | tips-and-patterns |
| Stale API after Python edits | PostToolUse auto-restart hook | notification-hooks |
| No health check on session start | SessionStart hooks | notification-hooks |

## How It Works

```
First run:
  Copy files into your project (e.g., docs/claude-setup/)
  "Read docs/claude-setup/SETUP.md and set up this project"
  -> 3-4 questions -> core files created -> done in 2 minutes

Later, when needed:
  "Read docs/claude-setup/research-tracking.md and set it up"
  -> detects existing config -> asks what's needed -> merges in

Or just ask naturally:
  "Set up research tracking"    -> Claude knows to read research-tracking.md
  "Add notifications"           -> reads notification-hooks.md
  "Add multi-agent support"     -> reads multi-agent-coordination.md

Re-run anytime:
  "Read SETUP.md again"         -> detects everything, only adds what's missing
```

## No-Copy Alternative

Don't want to copy files into your project? Use the self-contained prompt in **[project-setup-prompt-generic.md](project-setup-prompt-generic.md)** — paste it directly into a Claude Code session. It includes everything inline, no external files needed. Less modular but zero setup.

## Origin

These patterns emerged from building a script-to-previz pipeline involving:
- 350+ equivalent human work hours across 20+ sessions
- FastAPI + React + Unreal Engine 5 multi-stack development
- Three independent Claude Code agents coordinating via filesystem
- 14-hour sessions with multiple context compactions
- Real-time Telegram notifications for remote monitoring
- SQLite with 13 migrations, Three.js viewer, pipeline orchestration

General enough for any project. No attribution needed.
