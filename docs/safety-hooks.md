# Safety Hooks (PreToolUse Guards)

PreToolUse hooks that block dangerous operations before they execute. These are not suggestions or reminders -- they are hard blocks that prevent Claude from running commands that have caused real incidents.

## Why You Need This

Claude Code is powerful enough to run destructive commands. In long sessions, especially under pressure or after context compaction, Claude can:

- Kill processes by name (carpet-bombing every instance on the machine)
- Run `git reset --hard` or `git push --force` (destroying uncommitted work)
- Launch your app from the session (spawning competing processes)

These are not hypothetical. Each of these has caused real data loss or session crashes in production use.

## The Hook

Place this in your project-level `.claude/settings.json`:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "bash -c 'INPUT=$(cat); if echo \"$INPUT\" | grep -qiE \"taskkill[[:space:]]+/IM|Stop-Process[[:space:]]+-Name|Get-Process.*Stop-Process|pkill[[:space:]]|killall[[:space:]]\"; then echo \"{\\\"hookSpecificOutput\\\":{\\\"permissionDecision\\\":\\\"deny\\\",\\\"userMessage\\\":\\\"BLOCKED: Kill by PID only, never by name. List PIDs first with tasklist or ps.\\\"}}\"; exit 0; fi; if echo \"$INPUT\" | grep -qiE \"git[[:space:]]+reset[[:space:]]+--hard|git[[:space:]]+push[[:space:]]+--force|git[[:space:]]+push[[:space:]]+-f[[:space:]]|git[[:space:]]+clean[[:space:]]+-f\"; then echo \"{\\\"hookSpecificOutput\\\":{\\\"permissionDecision\\\":\\\"deny\\\",\\\"userMessage\\\":\\\"BLOCKED: Destructive git operation. Use safe alternatives (git stash, git revert, etc).\\\"}}\"; exit 0; fi; if echo \"$INPUT\" | grep -qiE \"npm[[:space:]]+start|npx[[:space:]]+electron-vite[[:space:]]+dev\"; then echo \"{\\\"hookSpecificOutput\\\":{\\\"permissionDecision\\\":\\\"deny\\\",\\\"userMessage\\\":\\\"BLOCKED: Do not launch the app from a Claude session. Tell the user to launch it manually.\\\"}}\"; exit 0; fi'",
            "timeout": 5
          }
        ]
      }
    ]
  }
}
```

## What It Blocks

### 1. Kill by Process Name

**Blocked commands:**
- `taskkill /IM electron.exe` (Windows)
- `Stop-Process -Name electron` (PowerShell)
- `Get-Process electron | Stop-Process` (PowerShell pipeline)
- `pkill python` (Linux/macOS)
- `killall node` (macOS)

**Why:** Killing by name hits every process with that name on the machine. If you have 3 instances of your app running, `taskkill /IM electron.exe` kills all of them. If a Python ML training job is running in another terminal, `pkill python` kills that too.

**Safe alternative:** Always kill by PID:

```bash
# Find the specific PID first
tasklist | grep electron        # Windows
ps aux | grep electron          # macOS/Linux

# Kill only that PID
taskkill //PID 12345 //T //F    # Windows (//T kills child processes)
kill -TERM 12345                # macOS/Linux
```

### 2. Destructive Git Operations

**Blocked commands:**
- `git reset --hard` (destroys uncommitted changes)
- `git push --force` / `git push -f` (overwrites remote history)
- `git clean -f` (deletes untracked files)

**Why:** In long sessions, Claude sometimes tries to "clean up" by resetting to a known state. This destroys uncommitted work. Force-pushing can overwrite teammates' commits.

**Safe alternatives:**

```bash
# Instead of git reset --hard
git stash                       # Save changes, can recover later
git stash pop                   # Restore later

# Instead of git push --force
git push --force-with-lease     # Only overwrites if no one else pushed

# Instead of git clean -f
git clean -n                    # Dry run first (shows what would be deleted)
```

### 3. App Launch from Session

**Blocked commands:**
- `npm start`
- `npx electron-vite dev`

**Why:** This is specific to Electron projects but the pattern applies broadly. Launching the app from within a Claude session can trigger session lifecycle hooks that spawn a competing Claude process, which then tries to absorb the current session. The result: your active session gets killed.

For non-Electron projects, you might block `python manage.py runserver` or `docker compose up` if launching them from the Claude session causes conflicts.

## Making It Readable: External Script

For complex safety checks, move the logic to a script file:

```json
{
  "matcher": "Bash",
  "hooks": [
    {
      "type": "command",
      "command": "\"$CLAUDE_PROJECT_DIR\"/.claude/hooks/safety-check.sh",
      "timeout": 5
    }
  ]
}
```

```bash
#!/usr/bin/env bash
# .claude/hooks/safety-check.sh
INPUT=$(cat)

deny() {
  echo "{\"hookSpecificOutput\":{\"permissionDecision\":\"deny\",\"userMessage\":\"$1\"}}"
  exit 0
}

# Block kill-by-name
if echo "$INPUT" | grep -qiE 'taskkill\s+/IM|Stop-Process\s+-Name|pkill\s|killall\s'; then
  deny "BLOCKED: Kill by PID only. List PIDs first."
fi

# Block destructive git
if echo "$INPUT" | grep -qiE 'git\s+reset\s+--hard|git\s+push\s+--force|git\s+push\s+-f\s|git\s+clean\s+-f'; then
  deny "BLOCKED: Destructive git operation. Use git stash or git revert."
fi

# Block app launch from session
if echo "$INPUT" | grep -qiE 'npm\s+start|yarn\s+dev|python\s+manage\.py\s+runserver'; then
  deny "BLOCKED: Do not launch the app from this session."
fi

# Additional project-specific blocks can go here
```

## Adding TypeScript Compile Reminders

A softer hook pattern: instead of blocking, remind Claude to verify after edits:

```json
{
  "matcher": "Edit",
  "hooks": [
    {
      "type": "command",
      "command": "bash -c 'INPUT=$(cat); if echo \"$INPUT\" | grep -qE \"\\.(ts|tsx)\\\"\"; then echo \"{\\\"hookSpecificOutput\\\":{\\\"additionalContext\\\":\\\"REMINDER: You just edited a TypeScript file. Run npx tsc --noEmit to verify it compiles.\\\"}}\"; fi'",
      "timeout": 5
    }
  ]
}
```

This does not block the edit -- it injects a reminder that Claude sees and acts on.

## The `permissionDecision` API

The key to making hooks block operations is the `permissionDecision` field:

| Value | Effect |
|-------|--------|
| `"allow"` | Explicitly allow (skips permission prompt) |
| `"deny"` | Block the operation entirely |
| (omitted) | Normal flow (may prompt user for permission) |

`userMessage` is shown to Claude as the reason for the block, so it can adjust its approach.

## Lessons Learned

**What works:**
- Hard blocks on kill-by-name -- this alone prevents the most common destructive incident
- `userMessage` that suggests the safe alternative -- Claude immediately tries the correct approach
- Separate matcher per tool type (`Bash`, `Edit`, `Write`) -- targeted, not blanket blocking
- Low timeout (5 seconds) -- safety hooks should be fast

**What does NOT work:**
- Blocking too many things -- Claude gets frustrated and finds workarounds
- Blocking without suggesting alternatives -- Claude retries the same blocked command
- Complex regex in inline bash -- hard to debug, move to an external script
- Forgetting to test the regex -- a bad pattern blocks legitimate commands or misses dangerous ones
