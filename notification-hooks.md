# Notification Hooks

Claude Code supports hooks — shell commands that run in response to events. These let you get notified when Claude needs attention, automate checks after edits, run health checks on session start, and protect against context loss.

## Auto-Setup

**For Claude**: If the user asked you to set up notification hooks, follow these steps:

1. **Check what exists**:
   - Read `~/.claude/settings.json` — are there existing hooks? Which types?
   - Read `.claude/settings.json` — project-level hooks?
   - Check for `~/.claude_credentials` — are TELEGRAM_BOT_TOKEN / TELEGRAM_CHAT_ID already set?
   - Report findings to user
2. **Ask the user**:
   - "Which notification channels do you want?" (Telegram / SMS / Desktop / None)
   - If Telegram: "Do you already have a bot token and chat ID?" If not, walk them through @BotFather setup
   - "Which hooks do you want?":
     - SessionStart (auto health check) — recommended
     - PostToolUse (auto type-check, pycache clear) — recommended for TS/Python projects
     - Notification: permission prompts (Claude needs approval) — recommended
     - Notification: task completion / idle (smart relay) — recommended
     - PreCompact (context compaction warning) — highly recommended
   - For existing hooks: "Keep existing hooks and add new ones, or replace?"
3. **Create/update files**:
   - MERGE new hooks into the appropriate settings.json (don't replace existing hooks)
   - Add credential entries to `~/.claude_credentials` if missing (with placeholder values and instructions)
4. **Test** — send a test Telegram/SMS to verify credentials work

---

## Two Levels of Hooks

Hooks can live in two places:

| File | Scope | Use for |
|------|-------|---------|
| `~/.claude/settings.json` | All projects (user-level) | Notifications, PreCompact, generic SessionStart |
| `.claude/settings.json` | This project only | Project-specific SessionStart, PostToolUse, project-specific automation |

Both files use the same format. Claude loads both and merges them.

## Hook Format

All hooks use this structure:

```json
{
  "hooks": {
    "<EventType>": [
      {
        "matcher": "<pattern>",
        "hooks": [
          {
            "type": "command",
            "command": "<shell command>"
          }
        ]
      }
    ]
  }
}
```

Event types: `SessionStart`, `PreToolUse`, `PostToolUse`, `Notification`, `PreCompact`, `Stop`.

The `matcher` field filters when the hook fires:
- `"startup"` — for SessionStart
- `""` (empty) — matches all events of that type (used for PreCompact)
- `"permission_prompt"` / `"idle_prompt"` — for Notification sub-events
- `"Edit|Write"` — regex match on tool name (for PostToolUse)

---

## Hook 1: SessionStart (project health check)

**What it does**: Runs automatically when a new conversation begins. Checks git status, server health, and tells Claude what to do.

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
            "command": "bash -c 'echo \"=== SESSION INIT ===\"; echo \"Project: MyProject\"; echo \"---\"; echo \"Git status:\"; cd /path/to/project && git status --short 2>/dev/null | head -15; echo \"---\"; echo \"API:\"; curl -sf http://localhost:8000/health 2>/dev/null && echo \" running\" || echo \" NOT running\"; echo \"Frontend:\"; curl -sf http://localhost:5173 >/dev/null 2>&1 && echo \" running\" || echo \" NOT running\"; echo \"---\"; echo \"ACTION: Read MEMORY.md for current state. Commit any uncommitted work first. Restart API if not running.\"'"
          }
        ]
      }
    ]
  }
}
```

**User-level `~/.claude/settings.json`** (generic, works for any project):
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

**Why it matters**: Claude starts every session with context about the project state. No more "where was I?" moments. The `ACTION:` line is critical — it tells Claude what to do with the information.

## Hook 2: PostToolUse (auto-checks after edits)

**What it does**: Fires after Claude uses a tool (Edit, Write, Bash, etc.). Use it to auto-run type checks, clear caches, or validate changes.

**Project-level `.claude/settings.json`:**

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
            "command": "FILE=$(echo \"$TOOL_INPUT\" | jq -r '.file_path // empty') && if echo \"$FILE\" | grep -qE '(api|pipeline|scripts)/.*\\.py$'; then find /path/to/project/api /path/to/project/pipeline -name '__pycache__' -type d -exec rm -rf {} + 2>/dev/null; echo '[hook] Cleared pycache — uvicorn --reload will pick up changes'; fi",
            "timeout": 10
          }
        ]
      }
    ]
  }
}
```

This example does two things after any Edit or Write:
1. If a `.tsx` or `.ts` file was edited, runs TypeScript type-check and shows the first 20 errors
2. If a Python file in `api/` or `pipeline/` was edited, clears `__pycache__` so uvicorn's `--reload` picks up changes

**Other ideas for PostToolUse:**
- Run ESLint after JS/TS edits
- Run `ruff check` after Python edits
- Validate JSON schema after config edits

## Hook 3: PreCompact (most important notification)

**What it does**: Fires before Claude's context window is compacted. Sends a notification AND echoes instructions that Claude sees.

**Why it matters**: Compaction can happen at any time during long sessions. Without this hook, Claude loses track of in-progress work. The echoed message tells Claude to save state before the wipe.

**User-level `~/.claude/settings.json`:**
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

With Telegram notification added:
```json
{
  "type": "command",
  "command": "bash -c 'source ~/.claude_credentials 2>/dev/null && curl -s \"https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/sendMessage\" -d \"chat_id=${TELEGRAM_CHAT_ID}\" -d \"text=[Claude] Context compacting soon. Saving state.\" > /dev/null 2>&1; echo \"COMPACTION IMMINENT. You MUST immediately: 1) Update MEMORY.md ACTIVE WORK. 2) Commit unsaved work. Do these NOW.\"'"
}
```

**The echo trick**: The hook's stdout is shown to Claude. By echoing "COMPACTION IMMINENT..." in the hook, Claude receives explicit instructions to save state. This is what makes it work — the notification alone isn't enough.

**CLAUDE.md instruction to pair with it:**
```markdown
When you see "COMPACTION IMMINENT", immediately:
1. Update MEMORY.md ACTIVE WORK with current state
2. Send a summary notification (write to /tmp/claude_telegram_msg.txt)
3. Commit and push unsaved work
```

## Hook 4: Smart Relay (idle notification)

**What it does**: When Claude finishes and goes idle, the hook checks for a message file. If it exists, sends it via Telegram and deletes the file.

**Why "smart"**: Claude only writes to `/tmp/claude_telegram_msg.txt` when there's genuinely useful info (milestones, errors, blockers). No file = no notification = no spam.

**User-level `~/.claude/settings.json`:**
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

**How Claude uses it** (add to CLAUDE.md):
```markdown
Write to `/tmp/claude_telegram_msg.txt` before going idle when there's useful info.
Only write for: milestones, errors, blockers, session summaries.
Every message MUST start with `[AgentName]` prefix.
No file = no notification = no spam.
```

**Example usage by Claude:**
```bash
cat > /tmp/claude_telegram_msg.txt << 'EOF'
[Pipeline] Render complete: test_42
Score: 8.2/10 (new best!)
Video: output/test_42_1920x1080.mp4
EOF
```

## Hook 5: Permission Prompt

**What it does**: Sends a notification whenever Claude needs permission to run a tool (file write, bash command, etc.).

**Why it matters**: If you're away from the terminal, Claude is blocked waiting for approval. This notifies you so you can come back and approve.

```json
{
  "matcher": "permission_prompt",
  "hooks": [
    {
      "type": "command",
      "command": "bash -c 'source ~/.claude_credentials 2>/dev/null && curl -s \"https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/sendMessage\" -d \"chat_id=${TELEGRAM_CHAT_ID}\" -d \"text=[Claude] Permission prompt - check terminal\" > /dev/null 2>&1 || true'"
    }
  ]
}
```

## Complete Example: `~/.claude/settings.json`

This is a real-world example combining all hooks:

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
    ],
    "PreCompact": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "bash -c 'source ~/.claude_credentials 2>/dev/null && curl -s \"https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/sendMessage\" -d \"chat_id=${TELEGRAM_CHAT_ID}\" -d \"text=[Claude] Context compacting soon. Saving state.\" > /dev/null 2>&1; echo \"COMPACTION IMMINENT. You MUST immediately: 1) Update MEMORY.md ACTIVE WORK. 2) Commit unsaved work. Do these NOW.\"'"
          }
        ]
      }
    ],
    "Notification": [
      {
        "matcher": "permission_prompt",
        "hooks": [
          {
            "type": "command",
            "command": "bash -c 'source ~/.claude_credentials 2>/dev/null && curl -s \"https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/sendMessage\" -d \"chat_id=${TELEGRAM_CHAT_ID}\" -d \"text=[Claude] Permission prompt - check terminal\" > /dev/null 2>&1 || true'"
          }
        ]
      },
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

## Setting Up Telegram

1. Message `@BotFather` on Telegram, send `/newbot`, follow prompts
2. Save the bot token (looks like `123456:ABC-DEF1234ghIkl-zyx57W2v1u123ew11`)
3. Send any message to your new bot
4. Get your chat ID:
   ```bash
   curl "https://api.telegram.org/bot<YOUR_TOKEN>/getUpdates" | python -m json.tool
   ```
   Look for `"chat": {"id": 123456789}`
5. Store in credentials file:
   ```bash
   echo 'export TELEGRAM_BOT_TOKEN="your-token"' >> ~/.claude_credentials
   echo 'export TELEGRAM_CHAT_ID="your-chat-id"' >> ~/.claude_credentials
   ```

## Alternative: SMS Notifications

For critical alerts (agent stopped, needs review, error), SMS provides a harder-to-miss channel.

Free Mobile (France) example:
```bash
curl "https://smsapi.free-mobile.fr/sendmsg?user=${SMS_USER}&pass=${SMS_PASS}&msg=Claude+needs+attention"
```

Other providers: Twilio, AWS SNS, or any SMS API.

**When to use SMS vs Telegram:**
- **Telegram**: Informational — scores, progress, session summaries
- **SMS**: Urgent — agent stopped, critical error, needs human decision

## Alternative: Desktop Notifications

For local-only projects where you're at the computer:

**macOS:**
```bash
osascript -e 'display notification "Claude needs attention" with title "Claude Code"'
```

**Linux:**
```bash
notify-send "Claude Code" "Claude needs attention"
```

**Windows (PowerShell via bash):**
```bash
powershell.exe -Command "[System.Reflection.Assembly]::LoadWithPartialName('System.Windows.Forms'); [System.Windows.Forms.MessageBox]::Show('Claude needs attention')"
```

## Credentials File Pattern

Store all API keys in one bash-sourceable file:

**~/.claude_credentials:**
```bash
export TELEGRAM_BOT_TOKEN="your-token"
export TELEGRAM_CHAT_ID="your-chat-id"
export SMS_USER="your-user"
export SMS_PASS="your-pass"
export HF_TOKEN="hf_xxx"
export OPENAI_API_KEY="sk-xxx"
# Add more as needed
```

Hooks source this file before using credentials. Never hardcode tokens in settings.json.
