# Research Tracking

Patterns for tracking what you've tried, what worked, what didn't, and what you learned — across sessions and across weeks. Applicable to any iterative project: ML experiments, feature development, debugging campaigns, performance tuning, or research.

## Auto-Setup

**For Claude**: If the user asked you to set up research tracking, follow these steps:

1. **Check what exists**:
   - `.claude/rules/banned-techniques.md` or similar "don't do this" file
   - Any `LEARNINGS*`, `TEST_TREE*`, `test_registry*`, `changelog*` files
   - `output/strategy/` or `docs/decisions/` or similar
   - CLAUDE.md sections about tracking, experiments, or dead ends
   - Report findings
2. **Ask the user**:
   - "What kind of work are you tracking?" (feature iterations, experiments, debugging, research, all of the above)
   - "Have you already hit dead ends you don't want to revisit? What categories?" (e.g., "Libraries", "Approaches", "Configurations")
   - "Do you want structured scoring/metrics for outputs?" If yes: "What metrics matter?"
   - "Where should tracking files go?" (default: `output/`)
   - For existing files: "Keep / Merge / Replace?"
3. **Create missing files**:
   - `.claude/rules/banned-techniques.md` with project-specific categories
   - `output/LEARNINGS.json` (empty template)
   - Add research tracking section to CLAUDE.md
4. **Explain the workflow** — tell the user how the pieces fit together

---

## Pattern 1: Banned Techniques / Dead Ends

The single most time-saving pattern for any iterative project. Without it, you (or Claude) will retry failed approaches across sessions.

**File: `.claude/rules/banned-techniques.md`**

```markdown
# Banned Techniques (proven harmful or dead — do NOT retry)

## [Category 1, e.g., Libraries]
- Library X — memory leak in production (tested 2026-03-05)
- Library Y — no TypeScript types, poor DX (evaluated 2026-03-06)

## [Category 2, e.g., Approaches]
- Approach A — works in dev but fails at scale (3 attempts, March 2026)

## Dead Ends (confirmed, don't revisit)
- Approach X: tried 5 times with different configs, never exceeded baseline
- Tool Y: incompatible with our stack
```

**Why `.claude/rules/`**: Files in this directory are auto-loaded into every Claude session for this project. Claude sees them without being asked — it literally cannot forget.

**Works for any project type:**
- **Web app**: banned npm packages, failed architectural patterns, deprecated APIs
- **ML pipeline**: failed model configs, broken hyperparameter combinations, incompatible tools
- **DevOps**: infrastructure approaches that don't scale, deployment strategies that failed
- **Research**: papers that didn't pan out, tools that don't deliver on promises

**Key rules:**
- Add entries as soon as something is confirmed dead. Don't wait.
- Include the date and context — it prevents "maybe we should try X again."
- Group by category for readability.
- Differentiate "harmful" (makes things worse) from "dead" (doesn't help) from "abandoned" (might work but not worth pursuing).

## Pattern 2: Research Registry

For projects with many iterations, maintain a structured log of what was tried.

**JSONL format (one JSON object per line):**

```jsonl
{"id": 1, "name": "baseline_v1", "date": "2026-03-01", "type": "feature", "params": {"framework": "react", "db": "postgres"}, "result": "working", "notes": "Initial setup"}
{"id": 2, "name": "switch_to_mongo", "date": "2026-03-02", "type": "experiment", "params": {"db": "mongodb"}, "result": "reverted", "notes": "Lost transaction support, not worth it"}
{"id": 3, "name": "add_caching", "date": "2026-03-03", "type": "optimization", "params": {"cache": "redis", "ttl": 300}, "result": "deployed", "notes": "2x faster API responses"}
```

**Why JSONL, not CSV**: Easier to append (one line per entry), supports nested objects, trivially parseable in any language, git-diff-friendly.

**Adapt the fields** to your project:
- ML: `params` = hyperparameters, `result` = score
- Web: `params` = tech choices, `result` = deployed/reverted
- DevOps: `params` = infra config, `result` = stable/rolled-back

## Pattern 3: LEARNINGS.json

Persistent cross-session learnings — things you've confirmed are true about your project:

```json
{
  "learnings": [
    {
      "id": 1,
      "category": "database",
      "finding": "PostgreSQL JSONB queries are fast enough — no need for a separate document store",
      "confidence": "high",
      "date_confirmed": "2026-03-05"
    },
    {
      "id": 2,
      "category": "deployment",
      "finding": "Cold starts on Lambda exceed 3s with our bundle size — need provisioned concurrency or switch to ECS",
      "confidence": "high",
      "date_confirmed": "2026-03-07"
    },
    {
      "id": 3,
      "category": "testing",
      "finding": "Integration tests catch 80% of prod bugs; unit tests on utils are low-value for this codebase",
      "confidence": "medium",
      "date_confirmed": "2026-03-10"
    }
  ]
}
```

CLAUDE.md instruction:
```markdown
Before trying a new approach, check LEARNINGS.json for relevant findings.
After confirming a finding across multiple instances, add it to LEARNINGS.json.
```

## Pattern 4: Decision Documents

When Claude (or you) makes a significant technical decision, save the reasoning:

```
output/decisions/
  2026-03-09_database-choice.md
  2026-03-12_auth-architecture.md
  2026-03-15_caching-strategy.md
```

Or `output/strategy/` — the name doesn't matter, the habit does.

**CLAUDE.md instruction:**
```markdown
Save significant decisions and analysis to `output/decisions/` with date-stamped filenames.
Never let important reasoning exist only in chat — it gets lost on compaction.
```

Why this matters:
- "Why did we choose X over Y?" comes up weeks later
- New team members (or new Claude sessions) need the context
- Decisions made during long sessions are the first thing lost during compaction

## Pattern 5: Progress Tree

For projects with branching work paths, maintain a visual tree:

```markdown
# Progress Tree

## Feature A: User Authentication
- A1: JWT tokens (working) -> SHIPPED
- A2: OAuth integration
  - A2.1: Google (working) -> SHIPPED
  - A2.2: GitHub (blocked — API rate limits) -> PAUSED
  - A2.3: Apple (not started)

## Feature B: Data Pipeline
- B1: batch processing (working, 10min/run) -> BASELINE
  - B1.1: add streaming (failed — memory issues) -> DEAD
  - B1.2: optimize batch (3min/run) -> CURRENT BEST
  - B1.3: switch to Spark -> NOT STARTED
```

This gives a bird's-eye view of what's been tried, what's working, and where the frontiers are.

## Putting It Together

```
project/
  .claude/rules/
    banned-techniques.md      # Auto-loaded, prevents retrying dead ends
  output/
    research_registry.jsonl    # Every significant attempt recorded
    LEARNINGS.json            # Confirmed findings
    PROGRESS_TREE.md          # Visual work/experiment tree
    decisions/                # Decision documents with reasoning
```

CLAUDE.md ties it together:
```markdown
## Research Tracking
- **Before trying an approach**: Check `banned-techniques.md` and `LEARNINGS.json`
- **After significant work**: Add to `research_registry.jsonl`, update `PROGRESS_TREE.md`
- **After confirming a finding**: Add to `LEARNINGS.json`
- **If approach is dead**: Add to `.claude/rules/banned-techniques.md`
- **Decisions/analysis**: Save to `output/decisions/` with date-stamped filenames
```
