# Notification Hooks

Claude Code supports hooks — shell commands that run in response to events. These let you get notified when Claude needs attention, finishes a task, or is about to lose context.

## Auto-Setup

**For Claude**: If the user asked you to set up notification hooks, follow these steps:

1. **Check what exists**:
   - Read `~/.claude/settings.json` — are there existing hooks? Which types?
   - Check for `~/.claude_credentials` — are TELEGRAM_BOT_TOKEN / TELEGRAM_CHAT_ID already set?
   - Check for any `.env` files with notification credentials
   - Report findings to user
2. **Ask the user**:
   - "Which notification channels do you want?" (Telegram / SMS / Desktop / None)
   - If Telegram: "Do you already have a bot token and chat ID?" If not, walk them through @BotFather setup
   - If SMS: "Which provider?" (Twilio, Free Mobile, AWS SNS) + "Do you have credentials?"
   - "Which events should trigger notifications?":
     - Permission prompts (Claude needs approval) — recommended
     - Task completion / idle (smart relay) — recommended
     - Context compaction warning — highly recommended
     - Custom project events (e.g., "render complete", "training done")
   - For existing hooks: "Keep existing hooks and add new ones, or replace?"
3. **Create/update files**:
   - MERGE new hooks into `~/.claude/settings.json` (don't replace existing hooks)
   - Add credential entries to `~/.claude_credentials` if missing (with placeholder values and instructions)
   - Add PreCompact hook if not present (always — this is critical even without notifications)
4. **Test** — send a test Telegram/SMS to verify credentials work

---

## Setup: ~/.claude/settings.json

All hooks go in `~/.claude/settings.json`. Here's a complete example with Telegram:

```json
{
  "hooks": {
    "PreCompact": [
      {
        "type": "command",
        "command": "bash -c 'source ~/.claude_credentials 2>/dev/null && curl -s \"https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/sendMessage\" --data-urlencode \"chat_id=${TELEGRAM_CHAT_ID}\" --data-urlencode \"text=[Claude] Context compacting soon. Saving state.\" > /dev/null 2>&1; echo \"COMPACTION IMMINENT. You MUST immediately: 1) Update MEMORY.md ACTIVE WORK. 2) Commit unsaved work. Do these NOW.\"'"
      }
    ],
    "Notification": [
      {
        "type": "command",
        "command": "bash -c 'source ~/.claude_credentials 2>/dev/null && MSG=$(cat /tmp/claude_telegram_msg.txt 2>/dev/null) && [ -n \"$MSG\" ] && curl -s \"https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/sendMessage\" --data-urlencode \"chat_id=${TELEGRAM_CHAT_ID}\" --data-urlencode \"text=$MSG\" > /dev/null 2>&1 && rm /tmp/claude_telegram_msg.txt; true'",
        "event": "idle_prompt"
      },
      {
        "type": "command",
        "command": "bash -c 'source ~/.claude_credentials 2>/dev/null && curl -s \"https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/sendMessage\" --data-urlencode \"chat_id=${TELEGRAM_CHAT_ID}\" --data-urlencode \"text=[Claude] Permission needed - check terminal\" > /dev/null 2>&1; true'",
        "event": "permission_prompt"
      }
    ]
  }
}
```

## Hook 1: Pre-Compact (most important)

**What it does**: Fires before Claude's context window is compacted. Sends a notification AND echoes instructions that Claude sees.

**Why it matters**: Compaction can happen at any time during long sessions. Without this hook, Claude loses track of in-progress work. The echoed message tells Claude to save state before the wipe.

**The echo trick**: The hook's stdout is shown to Claude. By echoing "COMPACTION IMMINENT..." in the hook, Claude receives explicit instructions to save state. This is what makes it work — the notification alone isn't enough.

**CLAUDE.md instruction to pair with it:**
```markdown
When you see "COMPACTION IMMINENT", immediately:
1. Update MEMORY.md ACTIVE WORK with current state
2. Send a summary notification (write to /tmp/claude_telegram_msg.txt)
3. Commit and push unsaved work
```

## Hook 2: Smart Relay (idle notification)

**What it does**: When Claude finishes and goes idle, the hook checks for a message file. If it exists, sends it via Telegram and deletes the file.

**Why "smart"**: Claude only writes to `/tmp/claude_telegram_msg.txt` when there's genuinely useful info (milestones, errors, blockers). No file = no notification = no spam.

**How Claude uses it** (add to CLAUDE.md):
```markdown
Write to `/tmp/claude_telegram_msg.txt` before going idle when there's useful info.
Only write for: milestones, errors, blockers, session summaries.
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

## Hook 3: Permission Prompt

**What it does**: Sends a notification whenever Claude needs permission to run a tool (file write, bash command, etc.).

**Why it matters**: If you're away from the terminal, Claude is blocked waiting for approval. This notifies you so you can come back and approve.

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
