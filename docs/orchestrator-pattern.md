# Orchestrator Pattern

The orchestrator pattern is how you scale Claude Code beyond a single session doing everything. The main conversation becomes a coordinator that delegates to specialized agents, never writing code inline.

## The Problem

In a single Claude session:
- Context fills up with implementation details
- Frontend CSS pollutes pipeline logic context
- No verification -- Claude says "done" without testing
- One task at a time, sequential
- No specialization -- same prompt handles all domains

## The Solution

```
You (human)
  |
  v
Orchestrator (main Claude session)
  |
  +-- Frontend Implementation Agent (writes React code)
  |     +-- UX Analyst Agent (reviews the UI)
  |
  +-- Backend Agent (writes FastAPI routes)
  |
  +-- Pipeline Agent (writes pipeline stages)
  |
  +-- QA Verifier Agent (verifies everything)
```

The orchestrator:
1. Receives the task from you
2. Decides which agent(s) to delegate to
3. Launches agents (sequentially or in parallel)
4. Collects results
5. Verifies via QA agent
6. Reports back to you

## Rules for the Orchestrator

Add this to your CLAUDE.md:

```markdown
## Orchestration Rules

- **DELEGATE**: Use subagents for all implementation work
- Your role is ORCHESTRATION -- launch agents, collect results, make decisions
- Never write feature code inline in the main conversation
- For tasks > 1 tool call, spawn an agent
- Always run QA Verifier after implementation agents commit
```

### When to Spawn an Agent

Rule of thumb from production use:

| Situation | Action |
|-----------|--------|
| Single file read/check | Do it inline |
| Quick git operation | Do it inline |
| Any code change (even "small") | Spawn implementation agent |
| Bug investigation + fix | Spawn agent |
| UI change | Spawn Frontend agent |
| API endpoint change | Spawn Backend agent |
| Multi-file refactor | Spawn agent with worktree isolation |

**The "more than 1 tool call" rule:** If the task will require more than one Edit/Write call, spawn an agent. This keeps the orchestrator's context clean.

## Agent Launch Pattern

The orchestrator describes what needs to be done, which agent to use, and what verification to expect:

```
Launch the Frontend Implementation agent:

Task: Add a search/filter bar to the props table in the QA sub-tab.
Context: Read `.claude/rules/frontend.md` and `.claude/rules/ux.md` first.
Requirements:
- Text search by prop name
- Filter by status: All / Meshed / No Mesh / Disabled
- Follow existing filter bar patterns in the app

Verification: tsc clean, Playwright screenshot of the filter in action.
```

## Parallel Agent Execution

For independent tasks, launch agents in parallel:

```
In parallel:
1. Frontend agent: Build the QA sub-tab grid view
2. Backend agent: Add the QA status endpoint
3. Pipeline agent: Implement the orientation analysis

Wait for all three, then run QA Verifier on all changes.
```

Use `isolation: worktree` in agent frontmatter to prevent file conflicts.

## Verification Chain

After every implementation agent commits:

```
Orchestrator
  1. Launch Frontend agent -> commits changes
  2. Launch QA Verifier on the commit
     - Verdict: PASS -> proceed
     - Verdict: FAIL -> send failure report back to Frontend agent
     - Verdict: UNVERIFIED -> note it, proceed with caution
  3. If FAIL: Frontend agent fixes -> QA Verifier re-checks
  4. Max 3 fix cycles, then escalate to human
```

## Agent Communication

### Via Orchestrator (Recommended)

The orchestrator passes results between agents:

```
1. Backend agent creates endpoint -> returns route + response shape
2. Orchestrator passes shape to Frontend agent
3. Frontend agent builds UI using the exact response shape
```

### Via Shared Files (For Complex Contracts)

For shared contracts that multiple agents reference:

```
docs/SCHEMA.md                # The canonical JSON schema
docs/AGENT_PIPELINE.md        # Pipeline agent's full context
docs/AGENT_FRONTEND.md        # Frontend agent's full context
```

Each agent reads their context file. Changes to the shared schema require orchestrator coordination.

### Via Filesystem Message Board (For Independent CLI Sessions)

If running multiple separate CLI sessions (not subagents):

```
agent_comms/
  FROM_pipeline_20260313_1400.md
  FROM_frontend_20260313_1500.md
```

See the agent comms protocol -- agents check this directory on session start.

## Real Workflow Example

A typical orchestrated workflow for "Add QA pipeline step":

```
Session Start:
  Orchestrator reads MEMORY.md, sees "P1: QA pipeline"

Step 1: Planning (inline)
  Orchestrator plans the feature, breaks into tasks

Step 2: Backend (spawn agent)
  Backend agent: Add /qa/status, /qa/run, /qa/reset endpoints
  -> Commits, API restarted

Step 3: Pipeline (spawn agent)
  Pipeline agent: Implement orientation analysis with Vision API
  -> Commits

Step 4: Frontend (spawn agent)
  Frontend agent: Build QA sub-tab with grid view, batch run, manual fix
  -> Commits, tsc clean

Step 5: QA Verification (spawn agent)
  QA Verifier: Test all endpoints with curl, screenshot UI with Playwright
  -> Verdict: PARTIAL (2 PASS, 1 UNVERIFIED -- Playwright couldn't click modal)

Step 6: Fix cycle (spawn agent)
  Frontend agent: Fix modal click issue (portal to document.body)
  -> Commits

Step 7: Final QA (spawn agent)
  QA Verifier: Re-test modal
  -> Verdict: PASS

Step 8: Update state
  Orchestrator updates MEMORY.md, commits, pushes
```

## Lessons Learned

**What works:**
- Orchestrator never writes code -- keeps context clean for decision-making
- QA Verifier as mandatory gate -- catches "works on my machine" issues
- Explicit task descriptions with context files -- agents start productive immediately
- Running Backend + Frontend in parallel with worktrees -- 2x throughput

**What does NOT work:**
- Trusting agents to verify their own work ("tsc clean" is not verification)
- Orchestrator doing "just one small fix" inline -- scope creep fills context
- Launching too many parallel agents (5+) -- coordination overhead exceeds benefit
- Skipping QA for "trivial" changes -- trivial changes break things regularly

**Rules of thumb:**
- 2-3 parallel agents is the sweet spot
- Max 3 fix cycles before escalating to human
- Always verify via QA agent, even for "obvious" changes
- The orchestrator's job is to THINK, not to CODE
