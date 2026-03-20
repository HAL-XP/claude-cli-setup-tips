# Claude Code Setup & Best Practices

Battle-tested patterns for getting the most out of Claude Code CLI, learned from 500+ hours of production development with 10 specialized agents, 740+ tests, and 50+ commits per session.

Framework-agnostic. Works for any project where Claude Code is your development partner.

Based on Claude Code v2.1.76+ (March 2026) with Opus 4.6 (1M context).

## Automated Setup with Claudeborn

Want to skip the manual setup? **[Claudeborn](https://github.com/HAL-XP/Claudeborn)** is an Electron wizard that applies these best practices automatically — just describe your project and it generates CLAUDE.md, hooks, rules, agents, devlog, and launch scripts in seconds.

<p align="center">
  <img src="https://raw.githubusercontent.com/HAL-XP/Claudeborn/master/screenshots/languages.png" alt="Claudeborn" width="600" />
</p>

## Quick Start

1. Read [Setup](docs/setup.md) for installation and first-run configuration
2. Read [Writing CLAUDE.md](docs/claude-md.md) for your project instruction file
3. Read [Hooks](docs/hooks.md) for the automation system that makes everything work
4. Pick the patterns you need from the guides below

## Documentation

### Core Setup

| Guide | What it covers |
|-------|---------------|
| [Setup](docs/setup.md) | Installation, credentials, first run, project structure |
| [Writing CLAUDE.md](docs/claude-md.md) | Effective project instructions, size management, imports |
| [Rules](docs/rules.md) | `.claude/rules/` directory, glob-scoped rules, domain separation |
| [Hooks](docs/hooks.md) | All 20+ hook event types, handler types, production examples |
| [Memory](docs/memory.md) | Auto-memory, MEMORY.md, topic files, cross-session persistence |

### Advanced Patterns

| Guide | What it covers |
|-------|---------------|
| [Agent Templates](docs/agents.md) | `.claude/agents/` directory, frontmatter, specialization, tool control |
| [Orchestrator Pattern](docs/orchestrator-pattern.md) | Main conversation delegates, never writes code inline |
| [Worktrees](docs/worktrees.md) | Parallel agent isolation with git worktrees |
| [MCP Servers](docs/mcp-servers.md) | Playwright, GitHub, custom MCP integrations |
| [Agent Teams](docs/agent-teams.md) | Experimental multi-agent collaboration (when to use, when to avoid) |
| [Telegram Notifications](docs/telegram-notifications.md) | Remote monitoring, smart relay, permission alerts |
| [Tips & Lessons Learned](docs/tips-and-tricks.md) | Anti-patterns, rules of thumb, platform-specific gotchas |

### Real Examples

| File | What it is |
|------|-----------|
| [CLAUDE.md example](examples/claude-md-example.md) | Real CLAUDE.md from a production project |
| [Agent template example](examples/agent-template-example.md) | Real agent template with frontmatter |
| [Rule example](examples/rule-example.md) | Real rule file with domain conventions |
| [Hooks config example](examples/hooks-example.json) | Real hooks configuration (user + project level) |

## What These Patterns Solve

| Problem | Pattern | Guide |
|---------|---------|-------|
| "Where was I?" after restarting | Auto-memory + SessionStart hooks | [Memory](docs/memory.md) |
| Context compaction destroys work | PreCompact hook saves state | [Hooks](docs/hooks.md) |
| Agent writes sloppy code | Specialized agents with verification gates | [Agents](docs/agents.md) |
| Agents step on each other's files | Worktree isolation + scope ownership | [Worktrees](docs/worktrees.md) |
| Retrying failed approaches | Banned techniques file (auto-loaded) | [Rules](docs/rules.md) |
| Claude blocked on permission, user AFK | Permission prompt notification | [Telegram](docs/telegram-notifications.md) |
| No idea if feature actually works | QA Verifier agent with 3-verdict system | [Orchestrator](docs/orchestrator-pattern.md) |
| Stale API after Python edits | PostToolUse auto-restart hook | [Hooks](docs/hooks.md) |
| CLAUDE.md is 500 lines and growing | Rules directory + imports | [Rules](docs/rules.md) |
| Multiple agents editing same file | Worktree isolation per agent | [Worktrees](docs/worktrees.md) |

## Architecture at a Glance

```
~/.claude/
  CLAUDE.md                    # Global user instructions (all projects)
  settings.json                # User-level hooks (PreCompact, Notification)
  agents/                      # User-level agent templates
  rules/                       # User-level rules
  projects/<hash>/memory/      # Auto-memory per project
    MEMORY.md                  # Main index (loaded every session)
    project_*.md               # Topic files (loaded on demand)

your-project/
  CLAUDE.md                    # Project instructions (checked into git)
  .claude/
    settings.json              # Project-level hooks (SessionStart, PostToolUse)
    settings.local.json        # Local overrides (gitignored)
    agents/                    # Project-level agent templates
      frontend-impl.md
      backend.md
      qa-verifier.md
    rules/                     # Auto-loaded rule files
      frontend.md
      python-api.md
      pipeline.md
      ux.md
      banned-techniques.md
  .mcp.json                    # MCP server configuration
```

## Origin

These patterns emerged from building a production multi-stack application involving:
- 500+ equivalent human work hours across 25+ sessions
- Python backend + React frontend + native engine integration
- 10 specialized agent templates coordinating via orchestrator pattern
- 740+ tests, 50+ commits per session
- Real-time Telegram notifications for remote monitoring
- Playwright MCP for autonomous browser testing

General enough for any project. No attribution needed.
