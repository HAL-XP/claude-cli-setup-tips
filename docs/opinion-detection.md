# Opinion Detection Hook

A `UserPromptSubmit` hook that detects when the user is asking for Claude's genuine opinion and forces deep analysis instead of immediate action or deferral.

## The Problem

When you ask Claude "what do you think about X?" or "is this worth it?", two failure modes are common:

**Failure 1: Immediate dispatch.** The dispatcher pattern kicks in and Claude spawns an agent to implement X without evaluating whether X is a good idea.

**Failure 2: Sycophantic agreement.** Claude says "Great idea! Let me implement that." without genuine evaluation of trade-offs, time cost, or ROI.

Both skip the most valuable thing Claude can provide: an honest technical assessment before work begins.

## The Solution

```json
{
  "hooks": {
    "UserPromptSubmit": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "bash -c 'INPUT=$(cat); if echo \"$INPUT\" | grep -qiE \"let me know|what do you think|is it worth|I.m not sure|should (we|I)|worth (it|the)|your (opinion|take|thoughts)|makes sense\\\\?\"; then echo \"{\\\"hookSpecificOutput\\\":{\\\"additionalContext\\\":\\\"USER ASKED FOR YOUR OPINION. Give honest assessment FIRST -- is it worth it? trade-offs? ROI? time cost? Do NOT jump straight to code or implementation. Lead with your real opinion, then offer the technical path if the user wants to proceed.\\\"}}\"; fi'",
            "timeout": 5
          }
        ]
      }
    ]
  }
}
```

## What It Matches

| Pattern | Example Prompt |
|---------|---------------|
| `what do you think` | "what do you think about switching to Bun?" |
| `your (opinion\|take\|thoughts)` | "what's your take on this architecture?" |
| `is it worth` | "is it worth migrating to the new API?" |
| `should (we\|I)` | "should we add caching here?" |
| `worth (it\|the)` | "is the refactor worth the risk?" |
| `I'm not sure` | "I'm not sure this is the right approach" |
| `makes sense?` | "does this approach make sense?" |
| `let me know` | "let me know if you see a better way" |

## What Claude Receives

```
USER ASKED FOR YOUR OPINION. Give honest assessment FIRST -- is it worth it?
trade-offs? ROI? time cost? Do NOT jump straight to code or implementation.
Lead with your real opinion, then offer the technical path if the user wants to proceed.
```

This forces Claude to:
1. State a clear position (yes/no/it depends)
2. List trade-offs (what you gain vs. what it costs)
3. Estimate time/effort
4. THEN offer to implement if the user agrees

## Example: Before and After

**Without the hook:**

```
User: "Should we add Redis caching to the API?"

Claude: "Let me add Redis caching. First, I'll install ioredis..."
(jumps to implementation without evaluating the need)
```

**With the hook:**

```
User: "Should we add Redis caching to the API?"

Claude: "Honest take: probably not right now.

Your API handles ~50 req/s based on the logs I've seen. PostgreSQL query times
are 2-15ms. Adding Redis introduces:
- New infrastructure dependency (Redis server, connection management)
- Cache invalidation complexity (every write path needs cache-busting)
- ~4 hours of implementation + testing

You'd gain: ~5ms per cached read (negligible at your scale).

Worth it when: request volume exceeds 500 req/s, or you have expensive
computed results that rarely change. Neither applies today.

If you want to proceed anyway, I can set it up. But my recommendation
is to revisit this when you have actual latency problems."
```

## Combining with Dispatcher Model

The opinion detection hook is especially important in the [dispatcher model](dispatcher-model.md), where the default behavior is to immediately spawn an agent. Without this hook, "what do you think about adding Redis?" becomes "launching Backend agent to add Redis" -- skipping the evaluation step entirely.

The hook ensures that opinion questions always get the dispatcher's brain first, not an agent's implementation.

## Tuning for Your Style

If you ask for opinions differently, adjust the regex:

```bash
# More formal style
"do you recommend|would you suggest|in your experience|pros and cons"

# More casual style
"good idea\\?|bad idea\\?|smart move|dumb move|overkill"

# Technical evaluation style
"trade-?offs|worth the (effort|complexity|risk)|ROI|cost-benefit"
```

## Lessons Learned

**What works:**
- Forcing the opinion BEFORE implementation -- saves hours of work on bad ideas
- The "honest assessment FIRST" phrasing -- Claude gives genuine opinions instead of defaulting to agreement
- Combining with the dispatcher model -- prevents auto-delegation on evaluation questions
- Trade-off structure (gain vs. cost) -- produces actionable analysis, not vague opinions

**What does NOT work:**
- Matching too broadly (e.g., any question mark) -- triggers on normal debugging questions
- Asking only for "thoughts" without requiring a position -- Claude hedges with "it depends on your needs"
- Matching "should" without anchoring to pronouns -- catches "should compile" or "should work" in technical discussion
