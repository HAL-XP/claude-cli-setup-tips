# Multi-Agent Coordination

When two or more Claude Code sessions work on the same project (e.g., one training models, another evaluating outputs), they need a way to communicate without the user copy-pasting between terminals.

## Auto-Setup

**For Claude**: If the user asked you to set up multi-agent coordination, follow these steps:

1. **Check what exists**:
   - Look for `agent_comms/`, `comms/`, or similar directories in the project
   - Check CLAUDE.md for existing inter-agent sections
   - Check if a watcher script already exists
   - Check if `/resume` skill already has a comms check step
   - Report findings
2. **Ask the user**:
   - "What are your agent names and roles?" (e.g., "Trainer — handles cloud GPU training" and "Pipeline — handles evaluation and scoring")
   - "Where should the shared message folder be?" (default: `agent_comms/` in project root)
   - "Do you want Telegram/notification alerts when a new message arrives?" (requires notification hooks)
   - For existing setup: "Keep current setup / Merge / Replace?"
3. **Create missing files**:
   - Comms directory with README.md (protocol docs)
   - Watcher script if notifications are wanted
   - Add inter-agent section to CLAUDE.md (for EACH agent — each needs to know about the others)
   - Add comms check step to `/resume` skill
   - Add scope separation section to CLAUDE.md
4. **Verify** — write a test message and confirm the watcher picks it up (if enabled)

---

## The Problem

- Agent A finishes training and wants to tell Agent B the checkpoint is ready
- Agent B finds a bug that affects Agent A's work
- The user is tired of being a human message bus between two CLI windows

## Pattern 1: Filesystem Message Board

Simple, no infrastructure needed. Both agents read/write files in a shared directory.

### Setup

```
project_root/agent_comms/
  README.md           # Protocol docs
  FROM_trainer_20260310_2300.md
  FROM_pipeline_20260310_2315.md
  .seen_messages      # Tracks which messages the watcher has processed
```

### Naming convention

```
FROM_<agent-name>_<YYYYMMDD>_<HHMM>.md
```

Examples:
- `FROM_trainer_20260310_2300.md`
- `FROM_pipeline_20260310_2315.md`
- `FROM_frontend_20260311_0900.md`

### Message format

```markdown
# From [Agent Name] — YYYY-MM-DD HH:MM TZ

## [Topic]

[Content — status updates, questions, results, requests]
```

### CLAUDE.md instruction (add to both agents)

```markdown
## Inter-Agent Communication
- **Shared message board**: `agent_comms/`
- Write messages as `FROM_<your-agent-name>_<YYYYMMDD_HHMM>.md`
- **On session start**: Check this folder for unread messages
- Agents: [Agent A] (purpose), [Agent B] (purpose)
```

## Pattern 2: Watcher Script with Notifications

A background script that polls the comms folder and sends notifications (Telegram, email, desktop) when new messages appear.

```python
"""Watch agent_comms/ folder and send notifications on new messages."""
import sys, os, time, urllib.request, urllib.parse
sys.stdout.reconfigure(encoding="utf-8", errors="replace")
# Line-buffered output for nohup
sys.stdout = open(sys.stdout.fileno(), mode='w', encoding='utf-8',
                  errors='replace', buffering=1)

COMMS_DIR = "path/to/agent_comms"
SEEN_FILE = os.path.join(COMMS_DIR, ".seen_messages")
POLL_INTERVAL = 30  # seconds

# Load your notification credentials
# source ~/.claude_credentials or hardcode for standalone use
TELEGRAM_BOT_TOKEN = os.environ.get("TELEGRAM_BOT_TOKEN", "")
TELEGRAM_CHAT_ID = os.environ.get("TELEGRAM_CHAT_ID", "")


def get_seen():
    if not os.path.exists(SEEN_FILE):
        return set()
    with open(SEEN_FILE) as f:
        return set(line.strip() for line in f if line.strip())


def mark_seen(filename):
    with open(SEEN_FILE, "a") as f:
        f.write(filename + "\n")


def send_telegram(text):
    if not TELEGRAM_BOT_TOKEN or not TELEGRAM_CHAT_ID:
        return
    try:
        data = urllib.parse.urlencode({
            "chat_id": TELEGRAM_CHAT_ID,
            "text": text[:4000]
        }).encode()
        req = urllib.request.Request(
            f"https://api.telegram.org/bot{TELEGRAM_BOT_TOKEN}/sendMessage",
            data=data
        )
        urllib.request.urlopen(req, timeout=10)
    except Exception as e:
        print(f"Telegram failed: {e}")


def main():
    print(f"Watching: {os.path.abspath(COMMS_DIR)}")
    print(f"Interval: {POLL_INTERVAL}s")

    while True:
        seen = get_seen()
        try:
            for f in sorted(os.listdir(COMMS_DIR)):
                if not f.startswith("FROM_") or not f.endswith(".md"):
                    continue
                if f in seen:
                    continue
                # New message
                ts = time.strftime("%H:%M:%S")
                print(f"\n  NEW MESSAGE: {f} ({ts})")
                filepath = os.path.join(COMMS_DIR, f)
                with open(filepath, encoding="utf-8") as fh:
                    content = fh.read()
                # First 3 lines as preview
                preview = "\n".join(content.strip().split("\n")[:3])
                send_telegram(f"[AgentComms] New message: {f}\n\n{preview}")
                mark_seen(f)
        except Exception as e:
            print(f"Error: {e}")
        time.sleep(POLL_INTERVAL)


if __name__ == "__main__":
    main()
```

Launch with:
```bash
nohup python agent_comms_watcher.py > /tmp/comms_watcher.log 2>&1 &
echo "Watcher PID: $!"
```

## Pattern 3: Scope Separation

When two agents touch the same codebase, clearly define ownership to prevent conflicts.

### Example scope doc (in each agent's CLAUDE.md)

```markdown
## Scope Separation
- **Agent A scope**: Cloud provisioning, training execution, checkpoint management
- **Agent B scope**: Evaluation, scoring, ComfyUI integration, strategy decisions
- **Shared**: agent_comms/ folder, output models directory
- **Don't duplicate**: Agent A's training details in Agent B's docs (point to theirs)
```

### Rules
- Each agent has its own CLAUDE.md section or separate instruction file
- Shared resources (models, datasets) have clear ownership for writes
- One agent doesn't modify the other's configs without communicating first
- Decisions that affect both agents go through the comms folder

## Pattern 4: Agent Name Prefix

When multiple agents send notifications, prefix every message with the agent name:

```
[Trainer] Step 8000 complete, checkpoint downloaded
[Pipeline] Eval results: struct 6.9, photo 8.0
[Research] Found new paper on conditioning architectures
```

This lets the user instantly identify which agent is talking. Set the name once at session start based on the task context.

Add to CLAUDE.md:
```markdown
Every notification MUST start with `[AgentName]` where AgentName describes this session's role.
Pick the name once at session start. Keep it consistent.
```
