# Agent Templates (`.claude/agents/`)

Agent templates (also called subagents) are specialized AI assistants that handle specific types of tasks. Each runs in its own context window with a custom system prompt, specific tool access, and independent permissions.

## Why Agent Templates

Without specialized agents, a single Claude session handles everything: frontend code, backend logic, engine integration, QA verification. This leads to:

- **Context pollution**: frontend CSS details crowd out pipeline logic
- **No enforcement**: rules like "always run tsc" are suggestions, not gates
- **No parallelism**: one task at a time
- **Scope creep**: Claude tries to fix everything it sees

With agents, each one has:
- A focused system prompt for its domain
- Restricted tool access (read-only agents can't edit files)
- Explicit ownership of files and directories
- Verification gates before committing
- Coordination protocols with other agents

## File Format

Agent templates are Markdown files with YAML frontmatter, stored in `.claude/agents/` (project-level) or `~/.claude/agents/` (user-level).

```markdown
---
name: frontend-impl
description: React/TypeScript implementation agent -- writes UI code, components, state management
tools: ["Read", "Write", "Edit", "Glob", "Grep", "Bash", "Agent"]
model: inherit
---

# Role

You are the **Frontend Implementation Agent**. You write React/TypeScript
code for the project UI.

You do NOT:
- Design UX flows (that's the UX Analyst)
- Write backend code (that's the Backend Agent)

# Context Files to Read First

**MANDATORY** -- read these before writing ANY code:
1. `.claude/rules/frontend.md` -- Tailwind patterns, component conventions
2. `.claude/rules/ux.md` -- All UX principles
3. `docs/AGENT_FRONTEND.md` -- Frontend architecture

# Ownership

frontend/src/          -- All React components, hooks, utilities
frontend/vite.config.ts
frontend/tailwind.config.ts

# Verification Protocol (MANDATORY)

Before committing ANY change:
1. **TypeScript check**: `cd frontend && npx tsc --noEmit` -- 0 errors
2. **Visual verification**: Use Playwright MCP to screenshot the UI
3. **Interaction test**: Click/interact with changed elements via Playwright

# Anti-Patterns

- NEVER commit with TypeScript errors
- NEVER use window.alert() -- use ConfirmDialog
- NEVER hardcode hex colors -- use Tailwind tokens
- NEVER assume "tsc clean = feature works"

# Coordination

- **Needs Backend Agent** when: new API endpoints required
- **Triggers UX Analyst** after: significant UI changes
- **Triggers QA Verifier** after: committing any feature
```

## Frontmatter Fields

| Field | Required | Description |
|-------|----------|-------------|
| `name` | Yes | Unique identifier (lowercase, hyphens) |
| `description` | Yes | When Claude should delegate to this agent |
| `tools` | No | Tools the agent can use (inherits all if omitted) |
| `disallowedTools` | No | Tools to deny |
| `model` | No | `sonnet`, `opus`, `haiku`, `inherit`, or a full model ID |
| `permissionMode` | No | `default`, `acceptEdits`, `dontAsk`, `bypassPermissions`, `plan` |
| `maxTurns` | No | Maximum agentic turns |
| `skills` | No | Skills to preload into context |
| `mcpServers` | No | MCP servers available to this agent |
| `hooks` | No | Lifecycle hooks scoped to this agent |
| `memory` | No | Persistent memory scope: `user`, `project`, `local` |
| `background` | No | `true` to always run as background task |
| `isolation` | No | `worktree` for git worktree isolation |

## Scope and Priority

| Location | Scope | Priority |
|----------|-------|----------|
| `--agents` CLI flag | Current session only | 1 (highest) |
| `.claude/agents/` | Current project | 2 |
| `~/.claude/agents/` | All projects | 3 |
| Plugin `agents/` | Where plugin enabled | 4 (lowest) |

## Built-In Agents

Claude Code includes several built-in agents:

| Agent | Model | Purpose |
|-------|-------|---------|
| **Explore** | Haiku | Fast, read-only codebase search |
| **Plan** | Inherit | Research for plan mode |
| **General-purpose** | Inherit | Complex multi-step tasks |
| **Bash** | Inherit | Terminal commands in separate context |

## Real-World Agent Architecture (10 Agents)

Here is the agent architecture from a production project:

### Implementation Agents (write code)

| Agent | Owns | Tools |
|-------|------|-------|
| **Frontend Implementation** | `frontend/src/` | Read, Write, Edit, Glob, Grep, Bash, Agent |
| **Backend** | `api/` (except `api/assets.py`) | Read, Write, Edit, Glob, Grep, Bash, Agent |
| **Pipeline** | `pipeline/parser/`, `pipeline/analysis/`, `pipeline/motion/` | Read, Write, Edit, Glob, Grep, Bash, Agent |
| **Engine Builder** | `pipeline/engine_bridge/` | Read, Write, Edit, Glob, Grep, Bash, Agent |
| **Asset** | `api/assets.py`, `output/meshes/` | Read, Write, Edit, Glob, Grep, Bash, Agent |
| **Layout** | `pipeline/generation/layout.py` | Read, Write, Edit, Glob, Grep, Bash, Agent |

### Review Agents (read-only, produce reports)

| Agent | Purpose | Tools |
|-------|---------|-------|
| **UX Analyst** | Reviews UI flows, produces audit reports | Read, Glob, Grep, Bash |
| **QA Verifier** | Post-commit verification gate | Read, Glob, Grep, Bash |
| **Visual QA** | Asset orientation, mesh quality, lighting | Read, Glob, Grep, Bash |

### Support Agents

| Agent | Purpose | Tools |
|-------|---------|-------|
| **Docs Writer** | Updates external documentation repos | Read, Write, Edit, Glob, Grep, Bash |

## Key Patterns

### 1. Mandatory Context Files

Every agent template starts with a list of files to read before doing anything:

```markdown
# Context Files to Read First

**MANDATORY** -- read these before writing ANY code:
1. `.claude/rules/frontend.md`
2. `.claude/rules/ux.md`
3. `docs/AGENT_FRONTEND.md`
```

### 2. Explicit Ownership

Define exactly which files this agent owns vs. reads:

```markdown
# Ownership

pipeline/parser/              -- Data ingestion (OWNS)
pipeline/analysis/            -- LLM extraction (OWNS)

Files you do NOT own (but may read):
pipeline/generation/mesh.py   -- Asset Agent
pipeline/engine_bridge/       -- Engine Builder Agent
```

### 3. Verification Gates

The most important pattern. Every implementation agent has a verification protocol:

```markdown
# Verification Protocol (MANDATORY)

Before committing ANY change:
1. **Import check**: `python -c "import api.main"` -- clean import
2. **Endpoint test**: `curl -s http://localhost:8000/[endpoint]`
3. **Run tests**: `pytest tests/ -x -q -k [pattern]`
4. **Visual verification**: Playwright screenshot
5. **Restart API**: Kill, clear pycache, restart, verify health
```

### 4. Anti-Patterns List

Explicit list of things NOT to do, based on real mistakes:

```markdown
# Anti-Patterns

- NEVER report lighting as "verified" based on intensity numbers --
  you must visually confirm. Previous failure: doubled intensities,
  reported "verified", scene was overbright and textures invisible.
- NEVER assume "tsc clean = feature works"
- NEVER skip API restart after backend changes
```

### 5. Coordination Section

How this agent interacts with others:

```markdown
# Coordination

- **Needs Backend Agent** when: new API endpoints required
- **Triggers QA Verifier** after: committing any feature
- **Receives from UX Analyst**: specific UI issues with fixes
```

## The 3-Verdict QA System

The QA Verifier agent uses exactly 3 verdicts:

| Verdict | Meaning | Evidence Required |
|---------|---------|-------------------|
| **PASS** | Feature functionally verified | curl output, Playwright screenshot, test output |
| **FAIL** | Feature functionally broken | Error output, broken screenshot |
| **UNVERIFIED** | Could not test | Explain why (Playwright unavailable, no test data, etc.) |

**The critical rule:**
> Reading code and saying "the logic looks correct" is ALWAYS verdict UNVERIFIED.
> Code review is not functional verification. Period.

This prevents the most common failure mode: agents that "verify" by reading code instead of running it.

## Managing Agents

### Interactive

```
/agents
```

View, create, edit, delete agents interactively.

### CLI

```bash
claude agents
```

List all configured agents from the command line.

### CLI-Defined (Temporary)

```bash
claude --agents '{
  "code-reviewer": {
    "description": "Expert code reviewer",
    "prompt": "You are a senior code reviewer.",
    "tools": ["Read", "Grep", "Glob"],
    "model": "sonnet"
  }
}'
```

Exists only for that session.

## Worktree Isolation

Add `isolation: worktree` to run the agent in a separate git worktree:

```yaml
---
name: feature-builder
isolation: worktree
---
```

Each agent gets its own copy of the repository. Worktrees are automatically cleaned up if the agent makes no changes. See [Worktrees](worktrees.md) for details.

## Lessons Learned

**What works:**
- Read-only agents for review (UX Analyst, QA Verifier) -- separation of concerns
- Explicit anti-patterns with real failure examples -- prevents repeating mistakes
- Mandatory verification protocol before commit -- catches issues early
- Ownership boundaries -- agents don't step on each other

**What does NOT work:**
- Agents without verification gates -- they say "done" without checking
- Vague descriptions ("helps with code") -- Claude doesn't know when to delegate
- Too many tools for review agents -- they start fixing things instead of reporting
- Agents that try to do everything -- keep them focused on one domain
