# Memory System

Claude Code has two complementary memory systems that carry knowledge across sessions:

1. **CLAUDE.md files** -- instructions you write (see [claude-md.md](claude-md.md))
2. **Auto memory** -- notes Claude writes itself based on corrections and patterns

Both are loaded at the start of every conversation.

## Auto Memory

Auto memory lets Claude accumulate knowledge across sessions without you writing anything. Claude saves notes as it works: build commands, debugging insights, architecture notes, code style preferences.

### Enabled by Default

Auto memory is on by default since Claude Code v2.1.59. Toggle with `/memory` or in settings:

```json
{ "autoMemoryEnabled": false }
```

### Custom Memory Directory

You can change where auto-memory files are stored with the `autoMemoryDirectory` setting:

```json
{ "autoMemoryDirectory": "/path/to/custom/memory/" }
```

This is useful if you want memory files in a shared or backed-up location. Default location remains `~/.claude/projects/<hash>/memory/`.

Added in v2.1.74.

### Storage Location

Each project gets its own memory directory:

```
~/.claude/projects/<project>/memory/
  MEMORY.md                    # Main index (first 200 lines loaded every session)
  debugging.md                 # Detailed notes on debugging patterns
  api-conventions.md           # API design decisions
  project_orientation_qa.md    # Topic-specific detail
```

The `<project>` path is derived from the git repository. All worktrees and subdirectories within the same repo share one auto memory directory.

### How It Works

- First 200 lines of `MEMORY.md` are loaded at session start
- Content beyond 200 lines is NOT loaded (Claude keeps it concise by splitting into topic files)
- Topic files are NOT loaded at startup -- Claude reads them on demand
- Claude reads and writes memory files during your session
- You see "Writing memory" or "Recalled memory" in the UI when this happens

### MEMORY.md as the Main Index

The primary file is `MEMORY.md`. Here is a production-style example:

```markdown
# Project Memory

## Current State (2026-03-16)
- **Active project**: demo-project
- **Last commit**: `ad1b3ff` -- session 25 changes
- **Next action**: Test character spawning in engine build

## Key Technical Findings
- R3F texture bug: <meshStandardMaterial map={texture}> doesn't trigger GPU update.
  Fix: key prop that changes on texture load forces material recreation.
- QA confidence threshold: 0.7 (was 0.5, too lenient)
- Dynamic lighting required: baked lighting renders gray without lightmap UVs

## User Preferences
- Always restart API after backend changes (user never does it)
- Move forward autonomously unless blocked
- Frontend agents must read UX rules first
- Telegram: results only, no fluff

## Architecture Decisions
- Content-aware staleness: hash upstream descriptions, skip regen when unchanged
- Two-tier review: Three.js (fast, geometric) first, engine (slow, visual) second

## Memory Files
- `project_qa_pipeline.md`: Batch QA plan (DONE)
- `project_asset_presets.md`: Character presets discovery
- `reference_playwright_mcp.md`: Playwright MCP setup reference
```

### Topic Files

For topics that need more detail than MEMORY.md can hold, create individual files:

```
~/.claude/projects/<hash>/memory/
  MEMORY.md                         # Main index (keep under 200 lines)
  project_gltf_transform.md         # Detailed topic file
  project_orientation_qa.md         # Another topic
  feedback_restart_servers.md       # User preference
  reference_playwright_mcp.md       # Reference info
```

**Naming conventions we use:**
- `project_*` -- project-specific technical details
- `feedback_*` -- user preferences and workflow choices
- `reference_*` -- reference documentation
- `plan_*` -- implementation plans

Reference these from MEMORY.md so Claude knows they exist.

### Rules for MEMORY.md

- **Keep under 200 lines** -- Claude loads this every session
- **Only store stable knowledge** -- confirmed across multiple interactions
- **Delete/update stale entries** -- wrong memory is worse than no memory
- **Update at every session end** -- this is the handoff to the next session

### Seeding MEMORY.md

For a new project, seed with this template:

```markdown
# Project Memory

## Current State
- (nothing yet -- update after first session)

## Key Technical Findings
- (populated as we discover things)

## Architecture Decisions
- (record important choices and WHY)

## User Preferences
- (workflow preferences, tool choices)
```

## Subagent Memory

Subagents can maintain their own persistent memory via the `memory` frontmatter field:

```yaml
---
name: code-reviewer
description: Reviews code for quality
memory: user
---
```

| Scope | Location | Use when |
|-------|----------|----------|
| `user` | `~/.claude/agent-memory/<name>/` | Learnings across all projects |
| `project` | `.claude/agent-memory/<name>/` | Project-specific (shareable via git) |
| `local` | `.claude/agent-memory-local/<name>/` | Project-specific (not in git) |

When memory is enabled, the subagent's system prompt includes instructions for reading/writing to its memory directory.

## View and Edit

Run `/memory` to:
- List all CLAUDE.md and rules files loaded in your session
- Toggle auto memory on/off
- Open the auto memory folder
- Select any file to open in your editor

## Devlog (`_devlog/`) (Optional)

For projects with many sessions per day, maintain a structured devlog at the project root. The underscore prefix keeps it sorted to the top in file explorers.

```
_devlog/
  summaries/          # Daily session summaries
    summary_YYYYMMDD.md
  hours/              # Human-equivalent hours tracking
    hours_YYYYMMDD.md
  decisions/          # Architecture decision records
    YYYYMMDD_topic.md
  experiments/        # Trial/spike results (adopt or kill)
    YYYYMMDD_topic.md
```

### Summaries

After each major piece of work, append to `_devlog/summaries/summary_YYYYMMDD.md`:

```markdown
## HH:MM -- Short title
- What was done
- Key decisions made
- Files changed
```

### Hours Tracking

Log human-equivalent hours to `_devlog/hours/hours_YYYYMMDD.md`:

```markdown
# Hours Log -- YYYY-MM-DD

| Task | Claude Time | Human Equiv. | Notes |
|------|-------------|--------------|-------|
| Built auth flow | 12min | 4h | New feature, OAuth integration |
| Fixed pagination bug | 3min | 30min | Edge case in cursor logic |
```

- "Human Equiv." = how long a skilled human developer would take for the same task
- One file per day, matching the summaries pattern
- Update at natural milestones, not after every tiny change

### Decisions

Record architecture choices in `_devlog/decisions/YYYYMMDD_topic.md`:

```markdown
# Auth Approach -- 2026-03-19

**Decision**: JWT with refresh tokens stored in httpOnly cookies.
**Alternatives considered**: Session-based auth, OAuth-only
**Why**: Stateless backend, no session store needed, mobile-friendly
```

### Experiments

Record trial results in `_devlog/experiments/YYYYMMDD_topic.md`:

```markdown
# LPIPS for Image Similarity -- 2026-03-10

**Goal**: Use LPIPS to detect duplicate/similar assets
**Result**: Requires GPU, results inconsistent across runs
**Verdict**: KILL -- added to banned-techniques.md
```

This complements MEMORY.md (current state) with a chronological record of what happened.

## Lessons Learned

**What works:**
- Session end protocol: always update MEMORY.md before ending
- PreCompact hook: forces Claude to save state before context loss
- Topic files for overflow: MEMORY.md stays lean, details in separate files
- User preferences in memory: Claude remembers your workflow choices

**What does NOT work:**
- Putting speculative conclusions in memory (from reading one file)
- Letting MEMORY.md grow past 200 lines (stops being loaded fully)
- Forgetting to clean up stale entries (wrong memory causes wrong decisions)
- Relying solely on chat memory without file persistence (lost on compaction)
