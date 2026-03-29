# Frustration Detection Hook

A `UserPromptSubmit` hook that detects user frustration and forces Claude to stop, analyze the root cause, and try a fundamentally different approach instead of retrying the same failed strategy.

## The Problem

When something breaks repeatedly, the natural interaction becomes:

1. Claude tries approach A. Fails.
2. User: "try again"
3. Claude tries approach A with slightly different parameters. Fails.
4. User: "this keeps failing, fix it"
5. Claude tries approach A one more time. Fails again.
6. User: "fuck, this is the same bug again"
7. Claude: "I apologize, let me try again..." (tries approach A a fourth time)

The pattern: Claude retries the same broken approach with minor variations. Each retry fills context with failure details, making it harder to think clearly. The user gets increasingly frustrated, and Claude gets increasingly apologetic but no more effective.

## The Solution

A hook that catches frustration signals and injects a hard stop:

```json
{
  "hooks": {
    "UserPromptSubmit": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "bash -c 'INPUT=$(cat); if echo \"$INPUT\" | grep -qiE \"fuck|fucking|pissed|same.*(issue|problem|bug).*again|keeps? (happening|failing|breaking)|how many times|tried this already|still broken|not working again\"; then echo \"{\\\"hookSpecificOutput\\\":{\\\"additionalContext\\\":\\\"FRUSTRATION DETECTED. STOP. Do NOT retry the same approach. Analyze: (1) What exact issue keeps recurring? (2) Why did previous fixes not work? (3) What assumption am I making that is wrong? (4) What is a fundamentally DIFFERENT approach? Take 30 seconds to think before acting. If you have tried the same fix twice, the approach is wrong -- not the parameters.\\\"}}\"; fi'",
            "timeout": 5
          }
        ]
      }
    ]
  }
}
```

## What It Matches

The regex catches common frustration patterns:

| Pattern | Matches |
|---------|---------|
| `fuck\|fucking` | Profanity (strong frustration signal) |
| `pissed` | Anger |
| `same.*(issue\|problem\|bug).*again` | "same issue again", "same problem again" |
| `keeps? (happening\|failing\|breaking)` | "keeps failing", "keep happening" |
| `how many times` | "how many times do I have to say this" |
| `tried this already` | Explicit repetition signal |
| `still broken` | Persistent failure |
| `not working again` | Regression |

You can tune the regex to match your communication style. Add patterns for your language if you work in a non-English environment.

## What Claude Sees

When the hook triggers, Claude receives this injected context:

```
FRUSTRATION DETECTED. STOP. Do NOT retry the same approach.
Analyze:
  (1) What exact issue keeps recurring?
  (2) Why did previous fixes not work?
  (3) What assumption am I making that is wrong?
  (4) What is a fundamentally DIFFERENT approach?
Take 30 seconds to think before acting.
If you have tried the same fix twice, the approach is wrong -- not the parameters.
```

This forces Claude to:
- Stop the retry loop
- Explicitly state what has been tried and why it failed
- Question its assumptions
- Propose a different strategy

## Extending: Severity Levels

You can differentiate between mild frustration (needs a pause) and strong frustration (needs a full reset):

```bash
#!/usr/bin/env bash
# .claude/hooks/frustration-detect.sh
INPUT=$(cat)

# Level 2: Strong frustration -- full root cause analysis
if echo "$INPUT" | grep -qiE "fuck|fucking|pissed|this is insane|I give up|waste of time"; then
  echo "{\"hookSpecificOutput\":{\"additionalContext\":\"STRONG FRUSTRATION. FULL STOP. Do NOT write any code yet. First: (1) List every approach tried so far. (2) Identify the common assumption across all attempts. (3) Propose 3 fundamentally different approaches. (4) Ask the user which direction to explore. You are in analysis mode, not implementation mode.\"}}"
  exit 0
fi

# Level 1: Mild frustration -- pause and reconsider
if echo "$INPUT" | grep -qiE "same.*(issue|problem|bug).*again|keeps? (happening|failing|breaking)|still (broken|not working)|tried this"; then
  echo "{\"hookSpecificOutput\":{\"additionalContext\":\"RECURRING ISSUE DETECTED. Before retrying: (1) What changed since the last attempt? (2) If nothing changed, why would the result be different? (3) Consider a different approach.\"}}"
  exit 0
fi
```

## Combining with Banned Techniques

After the frustration hook forces root cause analysis, save the failed approach to prevent future retries:

```markdown
# .claude/rules/banned-techniques.md

## Approaches That Do Not Work

### Terminal paste duplication
- Tried: Adding `e.preventDefault()` to keydown handler
- Tried: Debouncing paste events with 100ms timeout
- Root cause: Browser native paste AND event listener both fire
- Solution: Paste event listener must call `e.stopImmediatePropagation()`

### API timeout on large payloads
- Tried: Increasing timeout to 60s
- Tried: Chunking payload client-side
- Root cause: Server-side body parser limit, not timeout
- Solution: Increase express body-parser limit to 10mb
```

The banned techniques file is auto-loaded as a rule, so Claude sees it on every session start and knows not to retry dead ends.

## Lessons Learned

**What works:**
- The hook breaks the retry loop -- Claude genuinely changes strategy when forced to analyze
- The 4-question framework (what/why/assumption/different) produces useful analysis
- Combining with banned-techniques file prevents recurring failures across sessions
- Matching common frustration language catches real moments without false positives

**What does NOT work:**
- Matching too aggressively (e.g., "not working" alone) -- catches normal debugging conversation
- Injecting only "sorry" or "I understand" -- Claude needs a structured analysis framework, not empathy
- Skipping the root cause step -- without it, Claude just picks a random different approach instead of an informed one
