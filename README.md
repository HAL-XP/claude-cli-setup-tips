# Claude Code Project Setup Kit

Battle-tested patterns for getting the most out of Claude Code CLI, learned from 300+ hours of production development across multi-agent workflows.

Framework-agnostic — works for any project where Claude Code is your development partner.

## Quick Start

Tell Claude Code:
```
Read doc/claude-code-project-setup/SETUP.md and set up this project
```

Claude asks 3-4 essential questions, creates the core files, then shows you optional features you can enable anytime. Nothing is forced — you add features when you need them.

**Re-runnable**: Come back anytime and say "set up work tracking" or "add notifications" — it detects what exists and only adds what's missing.

**Individual guides**: Each guide works standalone too:
```
Read doc/claude-code-project-setup/session-continuity.md and set it up
```

## What's Inside

### Core (set up first)

| File | What it covers |
|------|---------------|
| **[SETUP.md](SETUP.md)** | Master installer — detects existing config, asks only what's needed, creates files, shows available add-ons. Start here. |
| **[session-continuity.md](session-continuity.md)** | MEMORY.md, /resume skill, pre-compact hooks, daily summaries, file-first output rule. The single biggest unlock — ensures no work is lost between sessions. |

### Add-ons (enable when you need them)

| File | What it covers | Best for |
|------|---------------|----------|
| **[notification-hooks.md](notification-hooks.md)** | Telegram/SMS/desktop notifications, smart relay (no spam), permission forwarding, pre-compact alerts. Full `settings.json` examples. | Remote work, AFK monitoring |
| **[multi-agent-coordination.md](multi-agent-coordination.md)** | Filesystem message board, watcher script with notifications, scope separation, `[AgentName]` prefix convention. | 2+ Claude sessions on same project |
| **[work-tracking.md](work-tracking.md)** | Banned techniques (auto-loaded dead ends), LEARNINGS.json, decision docs, progress trees, work registries. | Any iterative project — dev, ML, research |
| **[hours-tracking.md](hours-tracking.md)** | End-of-session time tracking — maps tasks to human-equivalent hours with multiplier. | ROI analysis, project planning |
| **[tips-and-patterns.md](tips-and-patterns.md)** | Long-running processes, background agents, credentials, Windows Unicode fixes, git workflow, custom skills, CLAUDE.md management. | Everyone — grab bag of essentials |

### Reference

| File | What it is |
|------|-----------|
| **[project-setup-prompt-generic.md](project-setup-prompt-generic.md)** | Alternative to SETUP.md — a prompt to paste into a new session. More verbose, asks more upfront questions. |

## What These Patterns Solve

| Problem | Pattern | Guide |
|---------|---------|-------|
| "Where was I?" after restarting | MEMORY.md + /resume skill | session-continuity |
| Context compaction destroys work | Pre-compact hook saves state | session-continuity |
| Retrying approaches that already failed | Banned techniques file (auto-loaded) | work-tracking |
| Copy-pasting between two Claude terminals | Filesystem message board | multi-agent-coordination |
| Claude blocked on permission, user is AFK | Permission prompt notification | notification-hooks |
| Important analysis lost after compaction | File-first output rule | session-continuity |
| No idea how much time Claude is saving | Hours tracking at session end | hours-tracking |
| Python crashes on Windows with Unicode | `sys.stdout.reconfigure` | tips-and-patterns |

## How It Works

```
First run:
  "Read SETUP.md and set up this project"
  → 3-4 questions → core files created → done in 2 minutes

Later, when needed:
  "Set up work tracking"     → reads work-tracking.md, merges into existing setup
  "Set up notifications"     → reads notification-hooks.md, walks through Telegram setup
  "Add multi-agent support"  → reads multi-agent-coordination.md, creates comms folder
  "Set up hours tracking"    → reads hours-tracking.md, creates rules + directory

Re-run anytime:
  "Read SETUP.md again"      → detects everything, only adds what's missing
```

## Origin

These patterns emerged from building an AI video pipeline involving:
- 300+ experiments with structured scoring
- Cloud GPU training with checkpoint management
- Two Claude agents coordinating via filesystem
- 14-hour sessions with multiple context compactions
- Real-time Telegram notifications for remote monitoring

General enough for any project. No attribution needed.
