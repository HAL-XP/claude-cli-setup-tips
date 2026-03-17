# Writing Effective CLAUDE.md Files

CLAUDE.md is a markdown file that gives Claude persistent instructions for your project. It is loaded into the context window at the start of every session.

## Where CLAUDE.md Files Live

| Scope | Location | Shared with |
|-------|----------|-------------|
| **Managed policy** | `/Library/Application Support/ClaudeCode/CLAUDE.md` (macOS), `/etc/claude-code/CLAUDE.md` (Linux), `C:\Program Files\ClaudeCode\CLAUDE.md` (Windows) | All users (org-wide) |
| **Project** | `./CLAUDE.md` or `./.claude/CLAUDE.md` | Team via source control |
| **User** | `~/.claude/CLAUDE.md` | Just you (all projects) |

More specific locations take precedence over broader ones. CLAUDE.md files in parent directories above your cwd are also loaded (walking up the tree).

CLAUDE.md files in subdirectories are loaded on demand when Claude reads files in those directories.

## Size Management

**Target: under 200 lines per CLAUDE.md file.**

Longer files consume more context and reduce adherence. This is the most common mistake people make.

**Strategies for keeping CLAUDE.md lean:**
- Move detailed rules to `.claude/rules/` files (auto-loaded, see [Rules](rules.md))
- Use `@path/to/file` imports for reference documentation
- Keep CLAUDE.md as an index pointing to detail files
- Archive completed project sections to `docs/`
- Use MEMORY.md (auto-memory) for evolving state, CLAUDE.md for stable rules

## Recommended Sections

A well-structured CLAUDE.md includes these sections:

### 1. Project Overview (5-10 lines)

```markdown
# My Project

One-paragraph description of what this project does, the tech stack,
and the current development phase.

## Stack
- **Backend**: FastAPI on localhost:8000
- **Frontend**: React + Vite on localhost:5173
- **Database**: PostgreSQL
```

### 2. Agent Contexts (if using multiple agents)

```markdown
## Agent Contexts

| Agent | Context File | Scope |
|---|---|---|
| **Pipeline** | `@docs/AGENT_PIPELINE.md` | Data processing, LLM extraction |
| **Frontend** | `@docs/AGENT_FRONTEND.md` | React UI, API routes |
| **Engine** | `@docs/AGENT_ENGINE.md` | Native engine integration |

The shared contract between all agents is: `docs/SCHEMA.md`
```

### 3. Key Conventions (10-20 lines)

```markdown
## Key Conventions

- All pipeline stages consume/produce JSON per `docs/SCHEMA.md`
- Location slugs: `INT. WAREHOUSE - NIGHT` -> `int_warehouse_night`
- Character IDs: `CHAR_001_ALICE` (stable across scenes)
- API keys in `~/.claude_credentials` (bash-sourceable)
- Never import engine-specific modules outside `pipeline/engine_bridge/`
```

### 4. Session Start Protocol

```markdown
## Session Start Protocol

When you see `SessionStart` hook output containing an `ACTION:` line,
execute those steps immediately before responding to the user:
1. Read `MEMORY.md` for current state
2. Commit any uncommitted work
3. Restart API if not running

Do not wait for the user to ask. Do not ask permission. Just do it.
```

### 5. What NOT To Do (5-10 lines)

```markdown
## What NOT To Do

- Do not commit large assets or credentials to git
- Do not modify `docs/SCHEMA.md` without updating all validators
- Do not call heavy APIs synchronously in route handlers
- Do not use `window.alert()` or `window.confirm()` in the frontend
```

### 6. Key Files Table

```markdown
## Key Files

| File | Purpose |
|------|---------|
| `docs/SCHEMA.md` | Canonical JSON schema (shared contract) |
| `.claude/rules/` | Domain-specific rules (auto-loaded) |
| `output/` | Pipeline output JSON |
```

## Import Syntax

CLAUDE.md files can import other files using `@path/to/file` syntax:

```markdown
See @README for project overview and @package.json for npm commands.

# Additional Instructions
- Git workflow: @docs/git-instructions.md
```

Relative paths resolve from the file containing the import. Max depth: 5 hops.

For personal preferences not checked into git:

```markdown
# Individual Preferences
- @~/.claude/my-project-instructions.md
```

## Generate with /init

Run `/init` in Claude Code to auto-generate a starter CLAUDE.md. Claude analyzes your codebase and creates a file with build commands, test instructions, and discovered conventions. If a CLAUDE.md already exists, `/init` suggests improvements rather than overwriting.

## View Loaded Instructions

Run `/memory` to see all CLAUDE.md and rules files currently loaded in your session. Select any file to open it in your editor.

## Lessons Learned

**What works:**
- Specific, verifiable instructions ("Use 2-space indentation" not "format code properly")
- Action-oriented session start protocol with explicit steps
- Keeping it under 200 lines forces you to prioritize

**What does NOT work:**
- Vague instructions ("be careful with the code")
- Contradicting rules across multiple CLAUDE.md files (Claude picks arbitrarily)
- Putting detailed domain rules in CLAUDE.md instead of `.claude/rules/`
- Trusting Claude to remember rules from long CLAUDE.md files (adherence drops with length)
