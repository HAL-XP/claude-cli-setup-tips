# Session Continuity

Claude Code sessions are ephemeral — context compaction or a new conversation means starting fresh. These patterns ensure no work is lost and every session picks up where the last one left off.

## Auto-Setup

**For Claude**: If the user asked you to set up session continuity, follow these steps:

1. **Check what exists** — scan for CLAUDE.md, MEMORY.md (in auto-memory dir), `.claude/skills/resume/`, `~/.claude/settings.json` (PreCompact hook), and any `*summary*` files. Report what you found and what's missing.
2. **Ask the user**:
   - "What's your timezone?" (for timestamps in ACTIVE WORK)
   - "Do you run background processes that need tracking between sessions?" (PIDs, logs)
   - "Do you use cloud GPU instances?" (add cloud check to /resume)
   - "Do you have multiple Claude agents on this project?" (add comms check to /resume)
   - For each existing file found: "Keep / Merge new content / Replace?"
3. **Create missing files** — MEMORY.md, /resume skill, PreCompact hook entry. Merge session continuity section into CLAUDE.md if it exists.
4. **Verify** — run `/resume` to confirm everything works.

---

## The Problem

Without session continuity:
- You restart a session and Claude has no idea what was in progress
- Background processes (training, builds, renders) are forgotten
- Decisions made 2 sessions ago get re-debated
- Dead-end approaches get retried

## Pattern 1: MEMORY.md with ACTIVE WORK

Claude Code has an auto-memory directory at `~/.claude/projects/<project-hash>/memory/`. Files here persist across conversations and MEMORY.md is loaded into every session's context.

### Structure

```markdown
# Project Memory

## ACTIVE WORK (updated YYYY-MM-DD, HH:MM TZ)
- **Current task**: What's in progress right now
- **Background processes**: PID 12345 doing X, log at /tmp/task_X.log
- **Cloud instances**: provider, ID, cost/hr, purpose
- **Pending decisions**: What's blocking, what data is needed
- **Next action**: Exactly what to do next

## Key Technical Findings
- Finding 1 (confirmed across N tests)
- Finding 2

## Architecture Decisions
- Decision + rationale + date

## User Preferences
- Workflow preferences
- Communication style
- Tool choices
```

### Rules
- **ACTIVE WORK gets updated every session end** — this is the handoff to the next session
- **Keep MEMORY.md under 200 lines** — only the first 200 are loaded. Use topic files for overflow (e.g., `memory/debugging.md`, `memory/architecture.md`) and link from MEMORY.md.
- **Only store stable knowledge** — things confirmed across multiple interactions. Don't write speculative conclusions from reading one file.
- **Delete/update stale entries** — wrong memory is worse than no memory.

### CLAUDE.md instruction to add

```markdown
## Session Continuity (MANDATORY)
- **On session start**: Check MEMORY.md "ACTIVE WORK", running PIDs, cloud instances, pending results.
- **On session end**: Update MEMORY.md with current task, PIDs, log paths, next action.
```

## Pattern 2: The /resume Skill

Create a custom skill that runs at the start of every session. It's a checklist that ensures nothing is missed.

**File: `.claude/skills/resume/SKILL.md`**

```markdown
---
description: Pick up where the last session left off
user_invocable: true
---

Do the following checks IN ORDER:

1. Read MEMORY.md "ACTIVE WORK" section
2. Check for running background processes: `ps aux | grep -E "python|node|train" | grep -v grep`
3. Check cloud instances if applicable (e.g., `vastai show instances`)
4. Check for pending results in output directories
5. Check inter-agent messages if multi-agent (see multi-agent-coordination.md)
6. Report status and ask what to work on

Do NOT start new work without completing these checks.
```

Users invoke with `/resume` at the start of every session.

## Pattern 3: Pre-Compact Hook

Context compaction can happen at any time when the conversation gets long. Without preparation, in-progress work is lost. The pre-compact hook gives Claude a warning.

**Add to `~/.claude/settings.json`:**

```json
{
  "hooks": {
    "PreCompact": [
      {
        "type": "command",
        "command": "echo 'COMPACTION IMMINENT. You MUST immediately: 1) Update MEMORY.md ACTIVE WORK section with current task, PIDs, log paths, next steps. 2) Commit and push unsaved work. Do these NOW before context is lost.'"
      }
    ]
  }
}
```

When compaction triggers, Claude sees this message and (if properly instructed in CLAUDE.md) saves state before context is wiped.

**CLAUDE.md instruction to add:**

```markdown
When you see the "COMPACTION IMMINENT" message, you MUST immediately:
1. Update MEMORY.md with current state
2. Commit/push unsaved work
3. Send a notification if configured
Do this BEFORE anything else.
```

## Pattern 4: Daily Summary Files

For projects with many sessions per day, maintain a running log:

```
project_output/_summary-YYYYMMDD.txt
```

Format:
```
Daily Summary — 2026-03-10

## Phase 1: [Description] (~10:00-14:00 CET)
- 10:00: Started X
- 12:00: Found bug Y, fixed with Z
- 14:00: Results: [scores/metrics]

## Phase 2: [Description] (~15:00-18:00 CET)
...
```

This complements MEMORY.md (which tracks current state) with a chronological record of what happened and when.

## Pattern 5: File-First Outputs

**Rule: anything important goes to a file, not just chat.**

Chat gets compacted. Strategy analysis, research findings, experiment results — if they only exist in the conversation, they're gone after compaction. Save them:

```
output/strategy/analysis_20260310.md
output/research_notes/research_notes_001.txt
output/LEARNINGS.json
```

Add to CLAUDE.md:
```markdown
- **Strategy outputs**: Save to `output/strategy/` with date-stamped filenames. Never let insights exist only in chat.
```
