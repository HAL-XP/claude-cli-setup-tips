# Setup

Installation, credentials, and first-run configuration for Claude Code CLI.

## Prerequisites

- **Node.js** 18+ (Claude Code is an npm package)
- **Git** (for worktrees, version control, and subagent isolation)
- **A Claude subscription** (Pro, Team, or Enterprise for full features; Max plan for Opus 4.6 with 1M context)

## Installation

```bash
npm install -g @anthropic-ai/claude-code
```

Verify:

```bash
claude --version
```

As of March 2026, the latest version is v2.1.76+.

## First Run

```bash
cd your-project
claude
```

On first run, Claude Code will:
1. Authenticate with your Anthropic account
2. Create `~/.claude/` directory for global config
3. Detect your project structure

### Generate a Starter CLAUDE.md

```
/init
```

This analyzes your codebase and generates a `CLAUDE.md` with build commands, test instructions, and project conventions. Refine from there.

## Project Structure

A well-configured Claude Code project looks like this:

```
your-project/
  CLAUDE.md                     # Project instructions (< 200 lines)
  .claude/
    settings.json               # Project hooks (SessionStart, PostToolUse)
    settings.local.json         # Local overrides (gitignored)
    agents/                     # Specialized agent templates
    rules/                      # Domain-specific rules (auto-loaded)
  .mcp.json                     # MCP server configuration
  .gitignore                    # Includes .env, *credentials*, __pycache__/
```

User-level config:

```
~/.claude/
  CLAUDE.md                     # Global instructions (all projects)
  settings.json                 # Global hooks, permissions
  agents/                       # User-level agent templates
  rules/                        # User-level rules
  projects/<hash>/memory/       # Auto-memory per project
```

## Credential Management

Store all API keys in one bash-sourceable file, never in the project directory:

**~/.claude_credentials:**

```bash
export ANTHROPIC_API_KEY="sk-ant-xxx"
export TELEGRAM_BOT_TOKEN="123456:ABC-DEF"
export TELEGRAM_CHAT_ID="123456789"
export HF_TOKEN="hf_xxx"
export FAL_KEY="fal-xxx"
export OPENAI_API_KEY="sk-xxx"
# Add more as needed
```

Usage in hooks:

```bash
source ~/.claude_credentials
curl -H "Authorization: Bearer $HF_TOKEN" ...
```

Usage in Python:

```python
import os
api_key = os.environ.get("FAL_KEY")
```

**CLAUDE.md instruction:**

```markdown
## Credentials
All API keys in `~/.claude_credentials` (bash-sourceable).
Load with: `source ~/.claude_credentials`
Do NOT look for a project-level `.env` — use `~/.claude_credentials` instead.
```

**Security rules:**
- Never commit credentials to git
- Add to `.gitignore`: `.env`, `*credentials*`
- Reference by variable name in CLAUDE.md, never paste actual values
- Hooks `source` the file at runtime so credentials never appear in `settings.json`

## Essential .gitignore Entries

```gitignore
.env
*credentials*
__pycache__/
*.pyc
node_modules/
.DS_Store
.claude/settings.local.json
.claude/agent-memory-local/
.claude/.pids
```

## Platform-Specific Notes

### Windows

**Unicode crash prevention** (mandatory for Python):

```python
import sys
sys.stdout.reconfigure(encoding="utf-8", errors="replace")
```

Every Python script on Windows needs this as the first executable line. The Windows console uses cp1252 encoding and any Unicode character (arrows, emoji, progress bars) will crash.

For subprocess launches:

```python
env = os.environ.copy()
env["PYTHONIOENCODING"] = "utf-8"
subprocess.Popen(cmd, env=env)
```

**Path handling in bash:**

Claude Code's bash on Windows uses forward slashes:

```bash
# Correct
ls "E:/Projects/my-project/src/"

# Wrong
ls "E:\Projects\my-project\src\"
```

**Process management:**

```bash
# Kill by PID (note double slash for Windows)
taskkill //PID 12345 //F

# Find processes
tasklist | grep python
```

### macOS / Linux

Generally no special considerations. Use standard Unix conventions.

## What to Set Up Next

After the basic setup:

1. **[CLAUDE.md](claude-md.md)** -- Write your project instructions
2. **[Rules](rules.md)** -- Split detailed rules into `.claude/rules/` files
3. **[Hooks](hooks.md)** -- Add SessionStart and PreCompact hooks (highest impact)
4. **[Memory](memory.md)** -- Seed your MEMORY.md with project state
5. **[Agents](agents.md)** -- Create specialized agent templates (when your project grows)
