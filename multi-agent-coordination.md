# Multi-Agent Coordination

When two or more **separate Claude Code CLI sessions** work on the same project (e.g., one building the backend, another building the frontend), they need a way to communicate without the user copy-pasting between terminals.

This guide is about **independent CLI agent instances** — separate terminal windows, each with their own conversation. Not about the built-in Agent tool subagents within a single conversation.

## Auto-Setup

**For Claude**: If the user asked you to set up multi-agent coordination, follow these steps:

1. **Check what exists**:
   - Look for `agent_comms/`, `comms/`, or similar directories in the project
   - Check CLAUDE.md for existing inter-agent sections
   - Check `.claude/rules/` for agent scope definitions
   - Check if SessionStart hooks already exist
   - Report findings
2. **Ask the user**:
   - "What are your agent names and roles?" (e.g., "Pipeline — handles backend + data pipeline" and "Frontend — handles React UI + API routes")
   - "Where should the shared message folder be?" (default: `agent_comms/` in project root)
   - "Do you want Telegram/notification alerts when a new message arrives?" (requires notification hooks)
   - For existing setup: "Keep current setup / Merge / Replace?"
3. **Create missing files**:
   - Comms directory with README.md (protocol docs)
   - Add inter-agent section to CLAUDE.md (for EACH agent — each needs to know about the others)
   - Add scope separation rules to `.claude/rules/`
   - Add comms check to SessionStart hook
4. **Verify** — write a test message and confirm the other agent can read it

---

## The Problem

- Agent A finishes a backend change and wants to tell Agent B the API contract changed
- Agent B finds a bug that affects Agent A's data model
- The user is tired of being a human message bus between two CLI windows

## Pattern 1: Filesystem Message Board

Simple, no infrastructure needed. Both agents read/write files in a shared directory.

### Setup

```
project_root/agent_comms/
  README.md           # Protocol docs (so each agent knows the rules)
  FROM_pipeline_20260310_2300.md
  FROM_frontend_20260310_2315.md
  FROM_ue5_20260311_0900.md
```

### Naming convention

```
FROM_<agent-name>_<YYYYMMDD>_<HHMM>.md
```

### Message format

```markdown
# From [Agent Name] — YYYY-MM-DD HH:MM TZ

## [Topic]

[Content — status updates, questions, results, requests]
```

### CLAUDE.md instruction (add to each agent's scope)

```markdown
## Inter-Agent Communication
- **Shared message board**: `agent_comms/`
- Write messages as `FROM_<your-agent-name>_<YYYYMMDD_HHMM>.md`
- **On session start**: Check this folder for unread messages from other agents
- Agents: [Pipeline] (backend, data), [Frontend] (React UI, API routes), [UE5] (Unreal Engine)
```

### Check messages on SessionStart

Add a comms check to the project-level SessionStart hook:

```json
{
  "hooks": {
    "SessionStart": [
      {
        "matcher": "startup",
        "hooks": [
          {
            "type": "command",
            "command": "bash -c 'echo \"=== SESSION INIT ===\"; echo \"Git status:\"; git status --short 2>/dev/null | head -15; echo \"---\"; MSGS=$(ls -1 agent_comms/FROM_*.md 2>/dev/null | wc -l | tr -d \" \"); echo \"Agent messages: $MSGS files\"; if [ \"$MSGS\" -gt 0 ]; then echo \"Latest:\"; ls -1t agent_comms/FROM_*.md 2>/dev/null | head -3; fi; echo \"---\"; echo \"ACTION: Read MEMORY.md. Check agent_comms/ for unread messages. Commit uncommitted work.\"'"
          }
        ]
      }
    ]
  }
}
```

## Pattern 2: Shared Context Directory

Beyond the message board, agents need a way to share structured knowledge — not just one-off messages, but ongoing state.

### Setup

Create a shared context directory that all agents read:

```
project_root/
  docs/
    AGENT_PIPELINE.md    # Pipeline agent's full context
    AGENT_FRONTEND.md    # Frontend agent's full context
    AGENT_UE5.md         # UE5 agent's full context
    SCENE_GRAPH_SCHEMA.md  # Shared contract (JSON schema, API types, etc.)
```

### How it works

Each agent's context file contains:
- What this agent owns (files, directories, responsibilities)
- Current status and recent changes
- Contracts with other agents (API shapes, file formats, shared schemas)
- Known issues that affect other agents

The main `CLAUDE.md` tells each agent which context file to load:

```markdown
## Agent Contexts

| Agent | Context File | Scope |
|---|---|---|
| **Pipeline** | `@docs/AGENT_PIPELINE.md` | Script parsing, LLM extraction, blocking, motion, camera |
| **Frontend** | `@docs/AGENT_FRONTEND.md` | FastAPI backend, React UI, job system, UX design |
| **UE5** | `@docs/AGENT_UE5.md` | Remote execution, scene building, MetaHuman, Sequencer |

The shared contract between all agents is JSON: `docs/SCENE_GRAPH_SCHEMA.md`
```

When starting a session for a specific agent, tell Claude: "You are the Frontend agent. Read `docs/AGENT_FRONTEND.md` for your context."

### Shared contract files

Define the boundary between agents explicitly:
- API response shapes (TypeScript interfaces or JSON schema)
- File formats (what each stage produces, what the next stage expects)
- Database schema (which agent owns which tables)
- Environment assumptions (ports, paths, required services)

When one agent changes the contract, they write to `agent_comms/` notifying the others.

## Pattern 3: Scope Separation

When two agents touch the same codebase, clearly define ownership to prevent conflicts.

### Example scope definition (in `.claude/rules/` or CLAUDE.md)

```markdown
## Scope Separation
- **Pipeline agent**: `pipeline/`, `scripts/`, `output/`, `docs/AGENT_PIPELINE.md`
- **Frontend agent**: `frontend/`, `api/`, `docs/AGENT_FRONTEND.md`
- **UE5 agent**: `pipeline/ue5_bridge/`, `docs/AGENT_UE5.md`
- **Shared (read-only for all)**: `docs/SCENE_GRAPH_SCHEMA.md`, `CLAUDE.md`
- **Shared (write by any)**: `agent_comms/`, `MEMORY.md`
```

### Rules
- Each agent has its own scope — don't modify files outside your scope without communicating first
- Shared resources have clear ownership for writes
- Decisions that affect multiple agents go through the comms folder
- If you need a change in another agent's scope, write a message requesting it

## Pattern 4: Agent Name Prefix

When multiple agents send notifications, prefix every message with the agent name:

```
[Pipeline] Stage complete: room_shell generated for 3 locations
[Frontend] Fix: category picker modal z-index issue
[UE5] Build complete: int_wooden_cabin_night loaded in editor
```

This lets the user instantly identify which agent is talking. Set the name once at session start based on the task context.

Add to CLAUDE.md:
```markdown
Every notification MUST start with `[AgentName]` where AgentName describes this session's role.
Pick the name once at session start. Keep it consistent.
```

## Practical Example: Three-Agent Project

A script-to-previz system with Pipeline, Frontend, and UE5 agents:

```
ScriptToUnreal/
  CLAUDE.md                      # Overview + agent table
  docs/
    AGENT_PIPELINE.md            # Pipeline scope, stage details
    AGENT_FRONTEND.md            # Frontend scope, API routes
    AGENT_UE5.md                 # UE5 scope, remote execution
    SCENE_GRAPH_SCHEMA.md        # Shared JSON contract
  .claude/rules/
    pipeline.md                  # Pipeline-specific rules
    frontend.md                  # Frontend-specific rules
    python-api.md                # API conventions
    ux.md                        # UX principles
  agent_comms/
    FROM_pipeline_20260313_1400.md   # "Scene graph schema updated, new field: front_yaw_offset"
    FROM_frontend_20260313_1500.md   # "Viewer now supports offset preview, ready for testing"
```

Each agent starts their session, the SessionStart hook fires, they check `agent_comms/` for new messages, read their context file, and proceed with their scoped work.

The user opens three terminal windows, starts Claude Code in each, and says:
- Terminal 1: "You are the Pipeline agent. Read docs/AGENT_PIPELINE.md."
- Terminal 2: "You are the Frontend agent. Read docs/AGENT_FRONTEND.md."
- Terminal 3: "You are the UE5 agent. Read docs/AGENT_UE5.md."

Each agent works independently within their scope, communicating through the filesystem.
