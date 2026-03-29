# Dispatcher Model

The dispatcher model is an evolution of the orchestrator pattern for teams that run Claude Code as a long-lived development partner. Instead of one session doing everything or manually spawning agents, the main session becomes a pure dispatcher: it routes tasks to agents, relays results, and never touches implementation itself.

## Why a Dispatcher

The orchestrator pattern (see [Orchestrator Pattern](orchestrator-pattern.md)) says "delegate to agents, don't write code inline." The dispatcher model takes this further:

| Orchestrator | Dispatcher |
|-------------|-----------|
| Plans the work, then delegates | Routes immediately, plans inside agents |
| Sometimes does "quick" inline fixes | Never does inline work beyond 1-liner reads |
| Context fills with coordination details | Context stays clean for routing decisions |
| Agent spawning is a tool | Agent spawning is the default action |

The key insight: every minute the dispatcher spends doing work is a minute its context fills with details that make future routing decisions worse. The dispatcher's value is in staying above the work, not in the work itself.

## The Rule

Add this to your CLAUDE.md:

```markdown
## YOU ARE A DISPATCHER (READ THIS FIRST)
- **1 liner** -> do it yourself. **Anything more** -> spawn an agent.
- **Missing agent type?** Build one.
- **Never do grunt work.** Your memory gets polluted, rules dilute, you drift.
- **Stay on comms** with the user. Agents do the work, you relay results.
```

"1 liner" means a single read, a single git command, a quick status check. The moment you think "I'll just edit this one file," you have crossed the line.

## Decision Matrix

| Task | Action | Why |
|------|--------|-----|
| `git status` | Do it yourself | Single command, no context cost |
| Read a file to answer a question | Do it yourself | Single read, answer, done |
| Fix a typo in one file | Do it yourself | 1 edit, no ripple effects |
| Change a component + update tests | Spawn agent | 2+ files, implementation detail |
| Bug investigation + fix | Spawn agent | Unknown scope, context-heavy |
| Visual UI change | Spawn agent + QA agent | Implementation + verification |
| "Quick refactor" | Spawn agent | "Quick" is always a lie |

## Agent Spawning with Context

The dispatcher provides context, not just commands. Agents are empowered to challenge the brief:

```markdown
Launch the Terminal Core agent:

Task: Fix the double-paste bug in TerminalPanel.tsx
Context: The paste event listener should block browser-native paste.
This has regressed before -- check git log for previous fixes.
Why: Users are getting duplicated text on Ctrl+V in the terminal.

Verification: tsc clean, manual test by pasting multiline text.
```

Compare this to the anti-pattern:

```markdown
Fix the paste bug in TerminalPanel.tsx.
```

The second version forces the agent to rediscover context that the dispatcher already knows.

## Parallelization

The dispatcher's superpower is launching multiple agents simultaneously:

```markdown
In parallel:
1. Terminal agent: Fix scrollback reconnect on renderer reload
2. 3D Visual agent: Add bloom threshold setting to PBR scene
3. Voice agent: Wire up personality sliders to TTS profile selection

Wait for all three, then QA each.
```

Sequential work is an anti-pattern. If two tasks are independent, they run in parallel. The dispatcher never asks "A or B?" -- it does both.

## What Happens Without It

Without the dispatcher rule, sessions drift into a predictable failure mode:

1. User asks for a feature
2. Session starts implementing (fills context with code details)
3. User asks a second question
4. Session tries to answer while holding implementation state
5. Context compacts, losing half the implementation
6. Session produces broken code from partial memory
7. User is frustrated, has to re-explain

With the dispatcher model:

1. User asks for a feature
2. Dispatcher spawns an implementation agent (context stays clean)
3. User asks a second question
4. Dispatcher answers immediately (context is not polluted)
5. Implementation agent finishes independently
6. Dispatcher relays the result

## Building Missing Agents

When the dispatcher encounters a task type it has no agent for, it builds one rather than doing the work inline:

```markdown
No agent template exists for database migrations.

Before implementing, I'll create one:
1. Write `.claude/agents/database-migration.md`
2. Define ownership: `migrations/`, `schema/`
3. Define verification: run migrations up + down, check schema state
4. Then spawn the new agent for the actual task
```

This front-loads 2 minutes of agent creation to save hours of context pollution.

## Communication Protocol

The dispatcher stays on comms with the user at all times:

- **Task received**: Acknowledge immediately, state the plan
- **Agent spawned**: Report which agent, what task, expected duration
- **Agent finished**: Relay results, highlight issues
- **Agent failed**: Report the failure, suggest next steps
- **Multiple agents**: Provide status updates as each completes

The user should never wonder "what is happening?" The dispatcher's job is to make the work visible.

## Lessons Learned

**What works:**
- Strict "1 liner" threshold -- the discipline pays off immediately
- Context briefs with "Why" for every agent spawn -- agents produce better work
- Parallel agent execution -- 2-3x throughput for independent tasks
- Building missing agents on demand -- the agent library grows organically

**What does NOT work:**
- "I'll just do this one quick thing" -- this is how dispatchers become implementers
- Spawning agents without context -- they waste turns rediscovering what you know
- More than 4 parallel agents -- coordination overhead exceeds benefit
- Dispatcher doing QA ("looks correct to me") -- always spawn a QA agent
