# Command Shortcuts (Hook-Based)

One-word commands that trigger specific behaviors via `UserPromptSubmit` hooks. Instead of typing "check the CI status and report," you type `ci`. Instead of "commit everything and push," you type `push`. The hook intercepts the prompt, injects context, and Claude acts accordingly.

## How It Works

A `UserPromptSubmit` hook receives the user's prompt before Claude sees it. The hook matches the prompt against keywords and injects `additionalContext` -- instructions that Claude follows as if the user had typed them.

```json
{
  "hooks": {
    "UserPromptSubmit": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "bash -c '<script that checks prompt and emits JSON>'",
            "timeout": 5
          }
        ]
      }
    ]
  }
}
```

The hook outputs JSON with `hookSpecificOutput.additionalContext` to inject instructions:

```json
{
  "hookSpecificOutput": {
    "additionalContext": "CI. Check the latest GitHub Actions CI run status and report."
  }
}
```

Claude sees this context prepended to the prompt and follows the instructions.

## The Hook Code

Place this in your project-level `.claude/settings.json`. This is a single hook with cascading `if` statements -- each keyword maps to an instruction set:

```json
{
  "hooks": {
    "UserPromptSubmit": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "bash -c 'PROMPT=\"$CLAUDE_USER_PROMPT\"; if echo \"$PROMPT\" | grep -qi \"^yours\"; then echo \"{\\\"hookSpecificOutput\\\":{\\\"additionalContext\\\":\\\"AGENT OWNS INSTANCE. Free to kill processes, rebuild, relaunch. User is not using the app.\\\"}}\"; exit 0; fi; if echo \"$PROMPT\" | grep -qi \"^mine\"; then echo \"{\\\"hookSpecificOutput\\\":{\\\"additionalContext\\\":\\\"HANDS OFF. Do NOT touch the running app. Do NOT kill processes. User is actively using it.\\\"}}\"; exit 0; fi; if echo \"$PROMPT\" | grep -qi \"^ci$\"; then echo \"{\\\"hookSpecificOutput\\\":{\\\"additionalContext\\\":\\\"CI. Check the latest GitHub Actions CI run status and report.\\\"}}\"; exit 0; fi; if echo \"$PROMPT\" | grep -qi \"^push$\"; then echo \"{\\\"hookSpecificOutput\\\":{\\\"additionalContext\\\":\\\"PUSH. Commit all staged changes and push to remote immediately.\\\"}}\"; exit 0; fi; if echo \"$PROMPT\" | grep -qi \"^test$\"; then echo \"{\\\"hookSpecificOutput\\\":{\\\"additionalContext\\\":\\\"TEST. Run the project test suite and report results.\\\"}}\"; exit 0; fi; if echo \"$PROMPT\" | grep -qi \"^nuke$\"; then echo \"{\\\"hookSpecificOutput\\\":{\\\"additionalContext\\\":\\\"NUKE. Kill all project processes, clear ALL caches, rebuild from scratch.\\\"}}\"; exit 0; fi; if echo \"$PROMPT\" | grep -qi \"^clean$\"; then echo \"{\\\"hookSpecificOutput\\\":{\\\"additionalContext\\\":\\\"CLEAN. Clear build caches only. No rebuild.\\\"}}\"; exit 0; fi; if echo \"$PROMPT\" | grep -qi \"^qa$\"; then echo \"{\\\"hookSpecificOutput\\\":{\\\"additionalContext\\\":\\\"QA. Spawn a QA agent to validate the last change.\\\"}}\"; exit 0; fi; if echo \"$PROMPT\" | grep -qi \"^ship$\"; then echo \"{\\\"hookSpecificOutput\\\":{\\\"additionalContext\\\":\\\"SHIP. Build, test, commit all changes, push to remote. Full release pipeline.\\\"}}\"; exit 0; fi; if echo \"$PROMPT\" | grep -qi \"^board$\"; then echo \"{\\\"hookSpecificOutput\\\":{\\\"additionalContext\\\":\\\"BOARD. Generate an interactive HTML dashboard of all TODOs and project state.\\\"}}\"; exit 0; fi; if echo \"$PROMPT\" | grep -qi \"^recap$\"; then echo \"{\\\"hookSpecificOutput\\\":{\\\"additionalContext\\\":\\\"RECAP. Summarize: key decisions, open questions, current direction, next steps.\\\"}}\"; exit 0; fi; if echo \"$PROMPT\" | grep -qi \"^perf$\"; then echo \"{\\\"hookSpecificOutput\\\":{\\\"additionalContext\\\":\\\"PERF. Run performance tests and report timing stats.\\\"}}\"; exit 0; fi; if echo \"$PROMPT\" | grep -qi \"^save state$\"; then echo \"{\\\"hookSpecificOutput\\\":{\\\"additionalContext\\\":\\\"SAVE STATE. Update MEMORY.md with current work, commit all changes, push.\\\"}}\"; exit 0; fi; if echo \"$PROMPT\" | grep -qi \"^relaunch$\"; then echo \"{\\\"hookSpecificOutput\\\":{\\\"additionalContext\\\":\\\"RELAUNCH. Kill running app, clear caches, rebuild, relaunch.\\\"}}\"; exit 0; fi; if echo \"$PROMPT\" | grep -qi \"^silent$\"; then echo \"{\\\"hookSpecificOutput\\\":{\\\"additionalContext\\\":\\\"SILENT. Stop sending notifications.\\\"}}\"; exit 0; fi; if echo \"$PROMPT\" | grep -qi \"^loud$\"; then echo \"{\\\"hookSpecificOutput\\\":{\\\"additionalContext\\\":\\\"LOUD. Resume sending notifications.\\\"}}\"; exit 0; fi; if echo \"$PROMPT\" | grep -qi \"^todo[: ]\"; then echo \"{\\\"hookSpecificOutput\\\":{\\\"additionalContext\\\":\\\"TODO. Add the following item to the project backlog.\\\"}}\"; exit 0; fi; if echo \"$PROMPT\" | grep -qi \"^queue[: ]\"; then echo \"{\\\"hookSpecificOutput\\\":{\\\"additionalContext\\\":\\\"QUEUE. Add to backlog as in-progress, then begin implementation immediately.\\\"}}\"; exit 0; fi;'",
            "timeout": 5
          }
        ]
      }
    ]
  }
}
```

## Command Reference

### Process Control

| Command | What it does |
|---------|-------------|
| `yours` | Tells Claude it owns the running app -- free to kill, rebuild, restart |
| `mine` | Tells Claude hands off -- user is actively using the app |
| `nuke` | Kill all project processes, clear all caches, rebuild from scratch |
| `clean` | Clear build caches only (no rebuild, no process killing) |
| `relaunch` | Kill app, clear caches, rebuild, relaunch |

### Development Flow

| Command | What it does |
|---------|-------------|
| `test` | Run the project test suite and report results |
| `qa` | Spawn a QA agent to verify the last change |
| `perf` | Run performance benchmarks and report stats |
| `push` | Stage, commit, and push all changes |
| `ship` | Full pipeline: build, test, commit, push |
| `ci` | Check latest CI run status on GitHub Actions |

### Project Management

| Command | What it does |
|---------|-------------|
| `board` | Generate an interactive HTML kanban dashboard |
| `recap` | Summarize current state: decisions, questions, direction |
| `save state` | Update MEMORY.md, commit everything, push |
| `todo: <text>` | Add an item to the project backlog |
| `queue: <text>` | Add to backlog AND start working on it immediately |

### Notification Control

| Command | What it does |
|---------|-------------|
| `silent` | Stop sending Telegram/notification messages |
| `loud` | Resume sending notifications |

## Adding Your Own Commands

To add a new command, append another `if` block to the chain:

```bash
if echo "$PROMPT" | grep -qi "^deploy$"; then
  echo "{\"hookSpecificOutput\":{\"additionalContext\":\"DEPLOY. Run deployment pipeline to staging environment.\"}}"
  exit 0
fi
```

Key rules:
- Use `^` anchor to match only at the start of the prompt
- Use `$` anchor for exact-match commands (like `^ci$`)
- Omit `$` for prefix commands that take arguments (like `^todo[: ]`)
- Always `exit 0` after matching to prevent falling through to other checks
- Keep the `additionalContext` concise but specific

## For Readability: External Script

If the inline bash gets unwieldy, move it to an external script:

```json
{
  "type": "command",
  "command": "\"$CLAUDE_PROJECT_DIR\"/.claude/hooks/shortcuts.sh",
  "timeout": 5
}
```

```bash
#!/usr/bin/env bash
# .claude/hooks/shortcuts.sh
PROMPT="$CLAUDE_USER_PROMPT"

emit() {
  echo "{\"hookSpecificOutput\":{\"additionalContext\":\"$1\"}}"
  exit 0
}

case "$PROMPT" in
  yours*) emit "AGENT OWNS INSTANCE. Free to kill, rebuild, relaunch." ;;
  mine*)  emit "HANDS OFF. Do NOT touch the running app." ;;
  ci)     emit "CI. Check latest GitHub Actions run and report." ;;
  push)   emit "PUSH. Commit all changes and push to remote." ;;
  test)   emit "TEST. Run test suite and report results." ;;
  nuke)   emit "NUKE. Kill processes, clear caches, rebuild." ;;
  ship)   emit "SHIP. Build, test, commit, push. Full pipeline." ;;
  *)      ;; # no match, pass through
esac
```

## Lessons Learned

**What works:**
- One-word commands for high-frequency actions -- typing `ci` 50 times a day saves significant time
- `yours`/`mine` ownership toggle -- prevents Claude from touching the running app when you are using it
- `todo:` prefix commands -- natural way to manage a backlog without leaving the terminal
- `save state` before stepping away -- ensures nothing is lost

**What does NOT work:**
- Too many commands -- if you can't remember them, they don't save time
- Ambiguous commands that overlap with normal conversation (e.g., "test this approach")
- Commands without clear `additionalContext` -- Claude guesses wrong
- Timeout too low -- 5 seconds is usually sufficient, but complex matching needs more
