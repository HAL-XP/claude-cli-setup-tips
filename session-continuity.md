# Session Continuity

Claude Code sessions are ephemeral — context compaction or a new conversation means starting fresh. These patterns ensure no work is lost and every session picks up where the last one left off.

## Auto-Setup

**For Claude**: If the user asked you to set up session continuity, follow these steps:

1. **Check what exists** — scan for CLAUDE.md, `.claude/settings.json` (SessionStart/PreCompact hooks), `~/.claude/settings.json` (user-level hooks), auto-memory files at `~/.claude/projects/*/memory/`, and any `*summary*` files. Report what you found and what's missing.
2. **Ask the user**:
   - "What's your timezone?" (for timestamps)
   - "Do you run background processes that need tracking between sessions?" (PIDs, logs)
   - "Do you have a backend server that should be checked on session start?" (URL for health check)
   - "Do you have multiple Claude agents on this project?" (add comms check)
   - For each existing file found: "Keep / Merge new content / Replace?"
3. **Create missing files** — MEMORY.md in auto-memory, SessionStart hook in `.claude/settings.json`, PreCompact hook in `~/.claude/settings.json`. Merge session continuity section into CLAUDE.md if it exists.
4. **Verify** — start a new conversation to confirm SessionStart hook fires correctly.

---

## The Problem

Without session continuity:
- You restart a session and Claude has no idea what was in progress
- Background processes (training, builds, renders) are forgotten
- Decisions made 2 sessions ago get re-debated
- Dead-end approaches get retried

## Pattern 1: Auto-Memory System

Claude Code has a built-in auto-memory system at `~/.claude/projects/<project-hash>/memory/`. Files here persist across conversations and are loaded into every session's context.

### MEMORY.md as the main index

The primary file is `MEMORY.md`. Claude reads it automatically at the start of every conversation.

```markdown
# Project Memory

## Current State (2026-03-13)
- **Active project**: minimaltest
- **Last commit**: `e156fd9` — session 20 changes
- **Next action**: Run orientation QA batch test

## Key Technical Findings
- Three.js version pinning: model-viewer uses three as peer dep — versions must match (0.x semver)
- UE5 Rotator constructor: Rotator(roll, pitch, yaw) — use named properties
- MOVABLE mobility required: Default STATIC renders gray without baked lightmaps

## Architecture Decisions
- Non-destructive front face system: preview offset in viewer, Save to bake into GLB
- Circulation clearance: 22 space categories with atmosphere modifiers

## User Preferences
- Always restart API after backend changes (user never does it)
- Move forward autonomously unless blocked
- Frontend agents must read UX rules first

## Memory Files
- `project_gltf_transform.md`: gltf-transform integration status
- `project_orientation_qa.md`: Batch orientation QA plan
```

### Additional memory files

For topics that need more detail than MEMORY.md can hold, create individual files in the same directory:

```
~/.claude/projects/<hash>/memory/
  MEMORY.md                      # Main index (keep under 200 lines)
  project_gltf_transform.md      # Detailed topic file
  project_orientation_qa.md      # Another topic
  plan_tab_restructure.md        # Completed plan (can be archived)
```

Reference these from MEMORY.md so Claude knows they exist.

### Rules
- **Keep MEMORY.md under 200 lines** — Claude loads this every session. Use topic files for overflow.
- **Only store stable knowledge** — things confirmed across multiple interactions. Don't write speculative conclusions from reading one file.
- **Delete/update stale entries** — wrong memory is worse than no memory.
- **Update at every session end** — this is the handoff to the next session.

### CLAUDE.md instruction to add

```markdown
## Session Continuity (MANDATORY)
- **On session start**: Check MEMORY.md current state, running PIDs, server health.
- **On session end**: Update MEMORY.md with current task, next action, any running processes.
- When you see the "COMPACTION IMMINENT" message from PreCompact hook, you MUST immediately:
  1. Update MEMORY.md with current state
  2. Commit/push unsaved work
  3. Send a notification if configured
  Do this BEFORE anything else.
```

## Pattern 2: SessionStart Hooks

Instead of relying on a custom `/resume` skill, use `SessionStart` hooks to automatically check project health every time a new conversation begins.

**Project-level `.claude/settings.json`:**

```json
{
  "hooks": {
    "SessionStart": [
      {
        "matcher": "startup",
        "hooks": [
          {
            "type": "command",
            "command": "bash -c 'echo \"=== SESSION INIT ===\"; echo \"Project: MyProject\"; echo \"---\"; echo \"Git status:\"; git status --short 2>/dev/null | head -15; echo \"---\"; echo \"API:\"; curl -sf http://localhost:8000/health 2>/dev/null && echo \" running\" || echo \" NOT running\"; echo \"---\"; echo \"ACTION: Read MEMORY.md for current state. Commit any uncommitted work first. Restart API if not running.\"'"
          }
        ]
      }
    ]
  }
}
```

**User-level `~/.claude/settings.json`** (applies to all projects):

```json
{
  "hooks": {
    "SessionStart": [
      {
        "matcher": "startup",
        "hooks": [
          {
            "type": "command",
            "command": "bash -c 'echo \"=== SESSION INIT ===\"; echo \"CWD: $(pwd)\"; echo \"Git branch: $(git branch --show-current 2>/dev/null || echo N/A)\"; echo \"Uncommitted: $(git status --short 2>/dev/null | wc -l | tr -d \" \") files\"; echo \"===\"'"
          }
        ]
      }
    ]
  }
}
```

The `ACTION:` line tells Claude what to do. Pair it with this CLAUDE.md section:

```markdown
## Session Start Protocol
When you see `SessionStart` hook output containing an `ACTION:` line, **execute those steps immediately before responding to the user**. Typical actions:
1. Read `MEMORY.md` for current state
2. Commit any uncommitted work (check git status, stage files, commit + push)
3. Restart API if not running
Do not wait for the user to ask. Do not ask permission. Just do it.
```

## Pattern 3: Pre-Compact Hook

Context compaction can happen at any time when the conversation gets long. Without preparation, in-progress work is lost. The pre-compact hook gives Claude a warning.

**Add to `~/.claude/settings.json`:**

```json
{
  "hooks": {
    "PreCompact": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "echo 'COMPACTION IMMINENT. You MUST immediately: 1) Update MEMORY.md ACTIVE WORK section with current task, PIDs, log paths, next steps. 2) Commit and push unsaved work. Do these NOW before context is lost.'"
          }
        ]
      }
    ]
  }
}
```

When compaction triggers, Claude sees this message and (if properly instructed in CLAUDE.md) saves state before context is wiped.

**The echo trick**: The hook's stdout is shown to Claude. By echoing "COMPACTION IMMINENT..." in the hook, Claude receives explicit instructions to save state. This is what makes it work — the notification alone isn't enough.

## Pattern 4: Daily Summary Files

For projects with many sessions per day, maintain a running log:

```
summary/summary-YYYYMMDD.md
```

Format:
```
# Daily Summary — 2026-03-10

## Session 1: Pipeline refactor (~10:00-14:00 CET)
- Commits: `abc1234`, `def5678`
- Completed circulation clearance system
- Found bug: Three.js version mismatch with model-viewer

## Session 2: Frontend QA (~15:00-18:00 CET)
- Commits: `ghi9012`
- Built orientation QA frontend
- Known issue: Vite cache needs clearing after npm install
```

This complements MEMORY.md (which tracks current state) with a chronological record of what happened and when.

## Pattern 5: File-First Outputs

**Rule: anything important goes to a file, not just chat.**

Chat gets compacted. Strategy analysis, research findings, experiment results — if they only exist in the conversation, they're gone after compaction. Save them:

```
output/strategy/analysis_20260310.md
output/decisions/2026-03-12_auth-architecture.md
output/LEARNINGS.json
```

Add to CLAUDE.md:
```markdown
- **Strategy outputs**: Save to `output/strategy/` with date-stamped filenames. Never let insights exist only in chat.
```
