# Hooks System

Hooks are event-driven automation scripts that execute in response to Claude Code lifecycle events. They are the backbone of session continuity, automatic checks, notifications, and quality gates.

## Two Levels of Hooks

| File | Scope | Use for |
|------|-------|---------|
| `~/.claude/settings.json` | All projects (user-level) | PreCompact, Notification, generic SessionStart |
| `.claude/settings.json` | This project only | Project-specific SessionStart, PostToolUse |
| `.claude/settings.local.json` | This project, not checked in | Local overrides |

Both files use the same format. Claude loads both and merges them.

## Hook Format

```json
{
  "hooks": {
    "<EventType>": [
      {
        "matcher": "<pattern>",
        "hooks": [
          {
            "type": "command",
            "command": "<shell command>",
            "timeout": 600,
            "async": false
          }
        ]
      }
    ]
  }
}
```

## All Hook Event Types

As of Claude Code v2.1.76, there are 20+ hook events across 6 categories:

### Lifecycle Events

| Event | Fires When | Matchers | Handler Types |
|-------|-----------|----------|---------------|
| **SessionStart** | Session begins/resumes | `startup`, `resume`, `clear`, `compact` | Command |
| **SessionEnd** | Session terminates | `clear`, `logout`, `prompt_input_exit`, `other` | Command |
| **InstructionsLoaded** | CLAUDE.md or rules file loads | (none) | Command |

### User Input Events

| Event | Fires When | Matchers | Can Block |
|-------|-----------|----------|-----------|
| **UserPromptSubmit** | User submits prompt | (none) | Yes |

### Tool Events

| Event | Fires When | Matchers | Can Block |
|-------|-----------|----------|-----------|
| **PreToolUse** | Before tool executes | Tool name (`Bash`, `Edit\|Write`, `mcp__.*`) | Yes |
| **PostToolUse** | After tool succeeds | Tool name | No |
| **PostToolUseFailure** | After tool fails | Tool name | No |
| **PermissionRequest** | Permission dialog appears | Tool name | Yes |

### Agent Events

| Event | Fires When | Matchers | Can Block |
|-------|-----------|----------|-----------|
| **PreCompact** | Before context compaction | `manual`, `auto` | No |
| **PostCompact** | After context compaction | `manual`, `auto` | No |
| **Stop** | Main agent finishes | (none) | Yes |
| **SubagentStart** | Subagent spawned | Agent type name | No |
| **SubagentStop** | Subagent finishes | Agent type name | Yes |
| **TeammateIdle** | Agent team member goes idle | (none) | Yes |
| **TaskCompleted** | Task marked complete | (none) | Yes |

### Configuration Events

| Event | Fires When | Matchers |
|-------|-----------|----------|
| **ConfigChange** | Config file changes | `user_settings`, `project_settings`, `skills` |
| **Notification** | Claude sends notification | `permission_prompt`, `idle_prompt`, `auth_success` |

### Isolation Events

| Event | Fires When |
|-------|-----------|
| **WorktreeCreate** | Creating isolated worktree |
| **WorktreeRemove** | Removing isolated worktree |
| **Elicitation** | MCP server requests user input |
| **ElicitationResult** | User responds to MCP request |

## Handler Types

### 1. Command Hooks (most common)

```json
{
  "type": "command",
  "command": "bash -c 'echo hello'",
  "timeout": 600,
  "async": false,
  "statusMessage": "Running check..."
}
```

Exit codes:
- `0`: Success (stdout parsed as JSON or shown to Claude)
- `2`: Blocking error (blocks the operation)
- Other: Non-blocking error

**The echo trick:** Hook stdout is shown to Claude. By echoing instructions in the hook, Claude receives explicit directions. This is what makes PreCompact work.

### 2. Prompt Hooks

Send the event data to a Claude model for evaluation:

```json
{
  "type": "prompt",
  "prompt": "Evaluate if this code change is safe: $ARGUMENTS",
  "model": "claude-haiku-4-5-sonnet",
  "timeout": 30
}
```

### 3. Agent Hooks

Spawn a full subagent with tool access for deep verification:

```json
{
  "type": "agent",
  "prompt": "Verify that this change meets security requirements",
  "timeout": 60
}
```

### 4. HTTP Hooks

POST event data to a URL endpoint:

```json
{
  "type": "http",
  "url": "https://localhost:8080/hook",
  "headers": { "Authorization": "Bearer $MY_TOKEN" },
  "timeout": 30
}
```

## The 5 Essential Hooks

These are the hooks that provide the most value. Set them up first.

### 1. SessionStart (Project Health Check)

The most immediately useful hook. Checks project state at the start of every conversation.

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
            "command": "bash -c 'echo \"=== SESSION INIT ===\"; echo \"Project: MyProject\"; echo \"---\"; echo \"Git status:\"; git status --short 2>/dev/null | head -15; echo \"---\"; echo \"API:\"; curl -sf http://localhost:8000/health 2>/dev/null && echo \" running\" || echo \" NOT running\"; echo \"Frontend:\"; curl -sf http://localhost:5173 >/dev/null 2>&1 && echo \" running\" || echo \" NOT running\"; echo \"---\"; echo \"ACTION: Read MEMORY.md for current state. Commit any uncommitted work. Restart API if not running.\"'"
          }
        ]
      }
    ]
  }
}
```

**User-level `~/.claude/settings.json`** (generic, all projects):

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

The `ACTION:` line at the end is critical. It tells Claude what to do. Pair with a CLAUDE.md section:

```markdown
## Session Start Protocol
When you see `SessionStart` hook output containing an `ACTION:` line,
execute those steps immediately before responding to the user.
```

### 2. PreCompact (Context Loss Prevention)

The most important hook for long sessions. Fires before Claude's context window is compacted.

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

With Telegram notification:

```json
{
  "type": "command",
  "command": "bash -c 'source ~/.claude_credentials 2>/dev/null && curl -s \"https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/sendMessage\" -d \"chat_id=${TELEGRAM_CHAT_ID}\" -d \"text=[Claude] Context compacting soon. Saving state.\" > /dev/null 2>&1; echo \"COMPACTION IMMINENT. You MUST immediately: 1) Update MEMORY.md ACTIVE WORK. 2) Commit unsaved work. Do these NOW.\"'"
}
```

### 3. PostToolUse (Auto-Checks After Edits)

Fires after Claude uses a tool. Use for automatic type-checking, cache clearing, linting.

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "FILE=$(echo \"$TOOL_INPUT\" | jq -r '.file_path // empty') && if echo \"$FILE\" | grep -qE '\\.(tsx|ts)$'; then cd /path/to/frontend && npx tsc --noEmit 2>&1 | head -20; fi"
          },
          {
            "type": "command",
            "command": "FILE=$(echo \"$TOOL_INPUT\" | jq -r '.file_path // empty') && if echo \"$FILE\" | grep -qE '(api|pipeline)/.*\\.py$'; then find /path/to/project -name '__pycache__' -type d -exec rm -rf {} + 2>/dev/null; echo '[hook] Cleared pycache'; fi",
            "timeout": 10
          }
        ]
      }
    ]
  }
}
```

### 4. Permission Prompt Notification

Sends a notification when Claude needs permission to run a tool (user is AFK).

```json
{
  "hooks": {
    "Notification": [
      {
        "matcher": "permission_prompt",
        "hooks": [
          {
            "type": "command",
            "command": "bash -c 'source ~/.claude_credentials 2>/dev/null && curl -s \"https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/sendMessage\" -d \"chat_id=${TELEGRAM_CHAT_ID}\" -d \"text=[Claude] Permission prompt - check terminal\" > /dev/null 2>&1 || true'"
          }
        ]
      }
    ]
  }
}
```

### 5. Smart Relay (Idle Notification)

When Claude finishes and goes idle, check for a message file. If it exists, send via Telegram.

```json
{
  "hooks": {
    "Notification": [
      {
        "matcher": "idle_prompt",
        "hooks": [
          {
            "type": "command",
            "command": "bash -c 'MSG_FILE=/tmp/claude_telegram_msg.txt; if [ -s \"$MSG_FILE\" ]; then source ~/.claude_credentials 2>/dev/null && MSG=$(cat \"$MSG_FILE\") && curl -s \"https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/sendMessage\" --data-urlencode \"chat_id=${TELEGRAM_CHAT_ID}\" --data-urlencode \"text=${MSG}\" > /dev/null 2>&1; rm -f \"$MSG_FILE\"; fi'"
          }
        ]
      }
    ]
  }
}
```

Claude writes to `/tmp/claude_telegram_msg.txt` when there is genuinely useful info. No file = no notification = no spam.

## Environment Variables in Hooks

### Setting Environment for the Session

SessionStart hooks can set environment variables for the entire session:

```bash
if [ -n "$CLAUDE_ENV_FILE" ]; then
  echo 'export NODE_ENV=production' >> "$CLAUDE_ENV_FILE"
fi
```

### Available Variables

- `$TOOL_INPUT` -- JSON of the tool's input (PostToolUse)
- `$TOOL_RESPONSE` -- JSON of the tool's response (PostToolUse)
- `$CLAUDE_PROJECT_DIR` -- Project directory path
- `$CLAUDE_ENV_FILE` -- Path to write env vars (SessionStart only)

### Reference Scripts by Path

```json
{
  "command": "\"$CLAUDE_PROJECT_DIR\"/.claude/hooks/my-script.sh"
}
```

## Advanced Features

### Async Hooks

Fire and forget, don't block Claude:

```json
{ "type": "command", "command": "script.sh", "async": true }
```

### Hook Deduplication

Identical handlers are automatically deduplicated (by command string or URL).

### Disable All Hooks

```json
{ "disableAllHooks": true }
```

### View Configured Hooks

Type `/hooks` in Claude Code to browse all active hooks.

## SessionStart Matcher Variants

| Matcher | When |
|---------|------|
| `startup` | Fresh new session |
| `resume` | Continuing a previous session |
| `clear` | Session was cleared |
| `compact` | Context was compacted |

You can run different hooks for each:

```json
{
  "SessionStart": [
    {
      "matcher": "startup",
      "hooks": [{ "type": "command", "command": "echo 'Fresh session'" }]
    },
    {
      "matcher": "resume",
      "hooks": [{ "type": "command", "command": "echo 'Resumed session'" }]
    }
  ]
}
```

## Lessons Learned

**What works:**
- The `ACTION:` line pattern in SessionStart -- Claude follows explicit instructions
- PreCompact echo trick -- Claude sees the echoed text and acts on it
- PostToolUse for TypeScript type-checking -- catches errors immediately
- Pycache clearing after Python edits -- uvicorn `--reload` picks up changes

**What does NOT work:**
- Complex multi-step scripts in hook commands -- keep them simple
- Hooks that take > 10 seconds -- use `async: true` or move to a background script
- Relying on notifications alone without the echo trick -- Claude needs to SEE the instruction
- SubagentStop hooks for Telegram notifications -- too spammy, removed in practice
