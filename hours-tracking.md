# Hours Tracking

Track the equivalent human work hours that Claude completes each session. Useful for: ROI analysis, project planning, communicating value to stakeholders, and keeping yourself honest about what's actually getting done.

## Auto-Setup

**For Claude**: If the user asked you to set up hours tracking, follow these steps:

1. **Check what exists**:
   - `.claude/rules/hours-tracking.md` — already configured?
   - `hours/` directory — existing logs?
   - CLAUDE.md — existing hours tracking section?
   - Report findings
2. **Ask the user**:
   - "What role title should I benchmark against?" (e.g., "senior full-stack developer", "senior ML engineer", "senior DevOps engineer") — this sets the baseline for human-hour estimates
   - "Do you want hours files tracked in git, or gitignored?" (default: tracked)
   - For existing setup: "Keep / Update role title / Replace?"
3. **Create missing files**:
   - `.claude/rules/hours-tracking.md` with the chosen role title
   - `hours/` directory
   - Add hours tracking section to CLAUDE.md
   - If gitignored is wanted, add `hours/` to `.gitignore`
4. **Done** — hours will be logged automatically at the end of each session

---

## Setup

### 1. Create the rule file

**File: `.claude/rules/hours-tracking.md`**

```markdown
# Human Work Hours Tracking

## Rule: Log equivalent human hours at end of each session

After completing work, before the session ends, create or update a daily hours log file at `hours/hours-YYYYMMDD.md` (same date convention as `summary/`).

## File Format

| Task | Commits | Human Hours | Notes |
|---|---|---|---|
| Short task description | `abc1234` | 4-6h | Brief context |
| Another task | `def5678`, `ghi9012` | 8-12h | Brief context |

**Total**: ~XXh (midpoint of ranges)
**Claude time**: ~Xh (actual wall clock)
**Multiplier**: ~XXx

## Estimation Guidelines

- Estimate how long a **senior full-stack developer** would take to do the same work
- Include: research, planning, coding, testing, debugging, code review
- Use ranges (e.g., "4-6h") to acknowledge uncertainty
- Group related commits into logical tasks
- Don't inflate — be honest and conservative
- The multiplier = total human hours / Claude wall clock time

## What Counts

- Feature implementation (backend + frontend)
- Bug fixes (include debugging time a human would spend)
- Refactoring / code reorganization
- Architecture plans and design docs
- UX audits and systematic improvements
- Pipeline/infrastructure work

## What Doesn't Count

- Trivial commits (typo fixes, comment changes)
- Git operations themselves
- Waiting for builds/deploys
```

This file is auto-loaded for the project (it's in `.claude/rules/`), so Claude will see it every session.

### 2. Add to CLAUDE.md

```markdown
## Hours Tracking
At the end of each session, create/update `hours/hours-YYYYMMDD.md` with tasks, commits,
and estimated equivalent human work hours (senior [your-role] engineer).
See `.claude/rules/hours-tracking.md` for format.
```

### 3. Create the directory

```bash
mkdir hours/
```

### 4. Add to .gitignore (optional)

If you don't want hours tracked in git, add `hours/` to `.gitignore`. Otherwise, tracking it in git gives you a history of productivity.

## Estimation Guidelines

### What role to benchmark against

Pick the role that matches your project:
- "senior full-stack developer" — web apps, APIs, databases
- "senior AI/ML engineer" — training, evaluation, model integration
- "senior DevOps engineer" — infrastructure, CI/CD, cloud
- "senior data engineer" — pipelines, ETL, data processing

### What to count

- **Research**: reading docs, papers, GitHub issues, model cards, forum posts
- **Planning**: architecture decisions, experiment design, strategy
- **Implementation**: writing code, scripts, configs, workflows
- **Debugging**: diagnosing failures, fixing bugs, iterating on solutions
- **Testing**: running tests, reviewing outputs, comparing results
- **Infrastructure**: cloud setup, deployment, model downloads, SSH tunneling
- **Communication**: writing docs, summaries, reports, messages
- **Monitoring**: active supervision of long-running tasks (not idle waiting)

### What NOT to count

- Pure idle waiting (a render running while nothing else happens)
- Time Claude spends polling/sleeping between checks
- Work that a human wouldn't need to do (e.g., Claude re-reading files it already processed)

### Estimation tips

- A 10-minute script fix that took 2 hours of debugging = **2h** human time (humans debug too)
- Multi-step debugging with dead ends counts fully — humans hit those too
- Research that leads nowhere still counts — a human would have spent that time
- Use **ranges** (e.g., "2-4h"), not point estimates. Add midpoint for totals.
- Be honest — don't inflate to impress, don't undercount to seem fast

## Example Output

```markdown
# Hours Log — 2026-03-10

| Task | Commits | Human Hours | Notes |
|---|---|---|---|
| Debug model loading (4 bugs found) | `abc1234` | 6-8h | Wrong model file, missing embeddings, disconnected conditioning, Unicode crash |
| Write evaluation scoring system | `def5678` | 3-4h | New reviewer prompt, side-by-side comparison, Excel integration |
| Set up inter-agent comms | `ghi9012` | 2-3h | Filesystem protocol, watcher script, Telegram notifications |
| Run and review 6 experiments | `jkl3456` | 2-3h | Render + review + score + compare for each |

**Total**: ~15.5h (midpoint of ranges)
**Claude time**: ~8h (with render waits)
**Multiplier**: ~1.9x
```

## Multiple Sessions Per Day

Append new session tables to the same daily file. Each gets its own header and totals. Add a daily grand total at the bottom.

## Multiplier Interpretation

| Multiplier | Meaning |
|-----------|---------|
| 1.0-1.5x | Claude saved some time but mostly did what you'd do |
| 1.5-3.0x | Solid productivity gain — parallelism, faster research, no fatigue |
| 3.0-5.0x | High leverage — Claude handled many parallel streams or deep debugging |
| 5.0x+ | Extraordinary — usually means 24/7 monitoring or massive parallelism |

The multiplier is Claude active time vs human equivalent, not wall clock time.
