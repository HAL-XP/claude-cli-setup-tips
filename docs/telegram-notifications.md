# Telegram Notifications

Telegram integration lets you monitor Claude Code remotely: permission prompts, context compaction warnings, milestone completions, and error alerts.

> **Update (March 2026):** Claude Code v2.1.80 introduced [Channels](channels.md) — an official two-way Telegram integration. Hooks (documented below) remain useful for automated system events, but Channels add interactive two-way communication. See [Channels](channels.md) for the modern approach, or use both together.

## Setting Up Telegram

### 1. Create a Bot

1. Message `@BotFather` on Telegram
2. Send `/newbot`, follow prompts
3. Save the bot token (looks like `123456:ABC-DEF1234ghIkl-zyx57W2v1u123ew11`)

### 2. Get Your Chat ID

1. Send any message to your new bot
2. Run:
   ```bash
   curl "https://api.telegram.org/bot<YOUR_TOKEN>/getUpdates" | python -m json.tool
   ```
3. Look for `"chat": {"id": 123456789}`

### 3. Store Credentials

```bash
echo 'export TELEGRAM_BOT_TOKEN="your-token"' >> ~/.claude_credentials
echo 'export TELEGRAM_CHAT_ID="your-chat-id"' >> ~/.claude_credentials
```

### 4. Test

```bash
source ~/.claude_credentials
curl -s "https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/sendMessage" \
  -d "chat_id=${TELEGRAM_CHAT_ID}" \
  -d "text=Test from Claude Code setup"
```

## Hook Configuration

Add these to `~/.claude/settings.json` (user-level, all projects):

### Permission Prompt Alert

When Claude needs approval and you are away from the terminal:

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

### Context Compaction Warning

When Claude is about to compact, saves state and notifies you:

```json
{
  "matcher": "",
  "hooks": [
    {
      "type": "command",
      "command": "bash -c 'source ~/.claude_credentials 2>/dev/null && curl -s \"https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/sendMessage\" -d \"chat_id=${TELEGRAM_CHAT_ID}\" -d \"text=[Claude] Context compacting soon. Saving state.\" > /dev/null 2>&1; echo \"COMPACTION IMMINENT. You MUST immediately: 1) Update MEMORY.md ACTIVE WORK. 2) Commit unsaved work. Do these NOW.\"'"
    }
  ]
}
```

### Smart Relay (Idle Notification)

Claude writes to a file when there is useful info. The hook sends it on idle:

```json
{
  "matcher": "idle_prompt",
  "hooks": [
    {
      "type": "command",
      "command": "bash -c 'MSG_FILE=/tmp/claude_telegram_msg.txt; if [ -s \"$MSG_FILE\" ]; then source ~/.claude_credentials 2>/dev/null && MSG=$(cat \"$MSG_FILE\") && curl -s \"https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/sendMessage\" --data-urlencode \"chat_id=${TELEGRAM_CHAT_ID}\" --data-urlencode \"text=${MSG}\" > /dev/null 2>&1; rm -f \"$MSG_FILE\"; fi'"
    }
  ]
}
```

## The Smart Relay Pattern

The smart relay is the key innovation: Claude only sends notifications when there is genuinely useful information.

**CLAUDE.md instruction:**

```markdown
Write to `/tmp/claude_telegram_msg.txt` before going idle when there's useful info.
Only write for: milestones completed, errors encountered, blockers found, session summaries.
Every message MUST start with `[AgentName]` prefix.
No file = no notification = no spam.
```

**How Claude uses it:**

```bash
cat > /tmp/claude_telegram_msg.txt << 'EOF'
[Pipeline] QA pipeline complete:
- 16 commits, all tests passing
- QA sub-tab with grid view, batch run, manual fix
- Layout blocked until all props passed/override
EOF
```

**Why this works:**
- No file = no notification = no spam
- Claude decides what is worth sending
- `[AgentName]` prefix lets you identify which session/agent sent it
- Only sent on idle (after Claude finishes work)

## Agent Name Convention

When multiple agents send notifications, each message starts with the agent name:

```
[Pipeline] Stage complete: data processed for 3 locations
[Frontend] Fix: category picker modal z-index issue
[Engine] Build complete: level_01 loaded in editor
```

Set the name once at session start based on the task context. Keep it consistent.

## Complete User-Level Settings

Real production `~/.claude/settings.json`:

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
            "command": "bash -c 'source ~/.claude_credentials 2>/dev/null && curl -s \"https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/sendMessage\" -d \"chat_id=${TELEGRAM_CHAT_ID}\" -d \"text=[Claude] Context compacting soon.\" > /dev/null 2>&1; echo \"COMPACTION IMMINENT. You MUST immediately: 1) Update MEMORY.md ACTIVE WORK. 2) Commit unsaved work. Do these NOW.\"'"
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

## Alternative Notification Channels

### SMS (for critical alerts)

Free Mobile (France):

```bash
curl "https://smsapi.free-mobile.fr/sendmsg?user=${SMS_USER}&pass=${SMS_PASS}&msg=Claude+needs+attention"
```

Other providers: Twilio, AWS SNS.

**When to use SMS vs Telegram:**
- **Telegram**: Informational (progress, summaries, milestones)
- **SMS**: Urgent (agent stopped, critical error, needs human decision)

### Desktop Notifications

**macOS:**
```bash
osascript -e 'display notification "Claude needs attention" with title "Claude Code"'
```

**Linux:**
```bash
notify-send "Claude Code" "Claude needs attention"
```

## Channels vs Hooks: When to Use What

Claude Code v2.1.80 introduced [Channels](channels.md) — a plugin-based system for two-way messaging with Telegram and Discord.

| Feature | Hooks (this page) | Channels |
|---------|-------------------|----------|
| **Direction** | One-way (Claude → you) | Two-way (you ↔ Claude) |
| **Auth** | Any (API key, claude.ai) | claude.ai login only |
| **Setup** | curl in settings.json | `--channels plugin:telegram@claude-plugins-official` |
| **Send commands from phone** | No | Yes |
| **Automated system alerts** | Yes (best for this) | Possible but hooks are simpler |
| **Works with all auth types** | Yes | No (claude.ai only) |

### Recommended Setup

Use **both** for the best experience:
- **Hooks** for automated alerts: permission prompts, compaction warnings, smart relay
- **Channel** for interactive control: send commands from your phone, ask questions, review output

The hook-based permission alert fires instantly when Claude is blocked. The channel gives you a Telegram chat where you can respond, send new instructions, and see Claude's replies — all from your phone.

## Lessons Learned

**What works:**
- Smart relay pattern -- no spam, only useful info
- `[AgentName]` prefix -- instantly know which agent sent it
- Permission prompt alerts -- unblocks Claude when you are away
- PreCompact notification -- gives you a heads-up that context is being compacted
- Combining hooks (automated alerts) + Channels (interactive) for full coverage

**What does NOT work:**
- SubagentStop notifications -- too frequent, removed in practice
- Sending "starting work" messages -- just noise
- Notifications without the echo trick -- Claude needs to SEE the instruction too
- Not using `--data-urlencode` for the message body -- special characters break the curl
