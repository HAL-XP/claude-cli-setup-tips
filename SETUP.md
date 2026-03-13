# Claude Code Project Setup — Master Installer

**For Claude**: The user asked you to set up their project with Claude Code best practices. This is a turnkey installer — follow these steps, ask questions where indicated, create all files yourself.

This setup is **re-runnable**. If the user comes back later wanting more features, re-read this file — Step 0 detects what exists and only adds what's missing.

---

## Step 0: Detect Existing Setup

Before asking ANY questions, silently scan for existing configuration:

```
Files to check:
- CLAUDE.md (project root) — existing project instructions?
- .claude/rules/*.md — existing rule files?
- .claude/settings.json — project-level hooks?
- .claude/skills/*/SKILL.md — existing skills?
- ~/.claude/CLAUDE.md — global user instructions?
- ~/.claude/settings.json — user-level hooks? Which types?
- ~/.claude/projects/*/memory/ — existing auto-memory files?
- ~/.claude_credentials or .env — existing credentials file?
- .gitignore — what's already ignored?
- agent_comms/ or similar — existing inter-agent setup?
- hours/ — existing hours tracking?
- output/ or similar — existing output structure?
```

Present what you found:

```
I scanned your project for existing Claude Code setup:

**Already configured:**
- [x] CLAUDE.md (N lines) — sections: [list headers]
- [x] .claude/rules/ — files: frontend.md, pipeline.md
- [x] ~/.claude/settings.json — hooks: PreCompact, Notification
- [x] etc.

**Not yet set up:**
- [ ] No .claude/settings.json (project-level hooks)
- [ ] No SessionStart hook
- [ ] No banned-techniques file
- [ ] etc.

For existing items, I'll keep them and merge new content alongside.
For missing items, I'll set them up now.
```

**Rules for existing config:**
- NEVER overwrite without explicit confirmation
- MERGE into existing CLAUDE.md — add sections, don't remove
- MERGE hooks into settings.json — don't replace the file
- If credentials exist, reference their location — don't create duplicates

---

## Step 1: Essential Questions (always ask)

Ask only these 3-4 questions upfront. Everything else can wait.

```
Let me set up Claude Code best practices for this project. Just a few essentials:

1. **Project name** and one-line description?
2. **Your timezone**? (e.g., "Paris/CET", "US Pacific/PST")
3. **Platform?** (I can probably detect this — [detected: Windows/Mac/Linux])
4. **Do you want notifications** when Claude needs attention? (Telegram recommended — works when you're AFK. Or: SMS, desktop, none)
```

That's it for now. I'll set up the core files, then list optional features you can enable anytime.

---

## Step 2: Create Core Files

Based on the answers + what's missing from Step 0, create these files. These are always useful regardless of project type.

### A. CLAUDE.md (project root)
If it exists, MERGE these sections. If not, create it.

Sections to include:
- **Project Goal**: name + description from answer 1
- **Session Continuity (MANDATORY)**: from `session-continuity.md`
- **Operational Notes**: timezone from answer 2, credentials location if known
- **Critical Rules**: platform-specific (Windows = Unicode fix, see `tips-and-patterns.md`)
- **Key Files**: table — populate as you create files
- **Git Workflow**: commit format with Co-Authored-By, push after milestones

**Keep CLAUDE.md lean** — under 150 lines. Detailed rules go in `.claude/rules/` files.

### B. `.claude/rules/` directory

Split rules into domain-specific files instead of putting everything in CLAUDE.md. Files in `.claude/rules/` are auto-loaded into every conversation for this project.

Recommended structure:
```
.claude/rules/
  banned-techniques.md   # Dead ends — never retry these
  <domain>.md            # One file per domain (e.g., frontend.md, api.md, pipeline.md)
```

Create `banned-techniques.md` with empty template:
```markdown
# Banned Techniques (proven harmful or dead — do NOT retry)
## [Add categories as you discover dead ends]
(none yet)
```

For projects with multiple domains (frontend + backend, pipeline + API, etc.), suggest creating domain-specific rule files. Example from a real project:
```
.claude/rules/
  frontend.md          # React/Tailwind/shadcn patterns, component conventions
  python-api.md        # FastAPI routes, job system, restart protocol
  pipeline.md          # Stage order, naming conventions, module ownership
  ux.md                # UX principles, state-driven UI, accessibility
  hours-tracking.md    # Hours log format and estimation guidelines
  banned-techniques.md # Dead ends
```

Each file stays focused. Claude loads all of them automatically — no need to reference them in CLAUDE.md.

### C. `.claude/settings.json` (project-level hooks)

Create a project-level settings file with a `SessionStart` hook that checks project health on every conversation start:

```json
{
  "hooks": {
    "SessionStart": [
      {
        "matcher": "startup",
        "hooks": [
          {
            "type": "command",
            "command": "bash -c 'echo \"=== SESSION INIT ===\"; echo \"Project: <project-name>\"; echo \"---\"; echo \"Git status:\"; git status --short 2>/dev/null | head -15; echo \"---\"; echo \"ACTION: Read MEMORY.md for current state. Commit any uncommitted work first.\"'"
          }
        ]
      }
    ]
  }
}
```

For projects with a backend server, add health checks:
```json
{
  "hooks": {
    "SessionStart": [
      {
        "matcher": "startup",
        "hooks": [
          {
            "type": "command",
            "command": "bash -c 'echo \"=== SESSION INIT ===\"; echo \"Git status:\"; git status --short 2>/dev/null | head -15; echo \"---\"; echo \"API:\"; curl -sf http://localhost:8000/health 2>/dev/null && echo \" running\" || echo \" NOT running\"; echo \"---\"; echo \"ACTION: Read MEMORY.md for current state. Commit any uncommitted work. Restart API if not running.\"'"
          }
        ]
      }
    ]
  }
}
```

The `ACTION:` line at the end is key — it tells Claude what to do with the information. See `session-continuity.md` for the CLAUDE.md instruction that pairs with this.

### D. Auto-memory MEMORY.md

Claude Code's auto-memory lives at `~/.claude/projects/<project-hash>/memory/`. Claude creates and manages this automatically. You can seed it with an initial MEMORY.md:

```markdown
# Project Memory

## Current State
- (nothing yet — update this every session end)

## Key Technical Findings
- (populated as we discover things)

## Architecture Decisions
- (record important choices and WHY)

## User Preferences
- (workflow preferences, tool choices, communication style)
```

### E. `.gitignore` additions
Ensure these entries exist (don't duplicate):
```
.env
*credentials*
**/temp/
__pycache__/
*.pyc
```

### F. PreCompact hook (always — this is critical)
Even without notifications, add the echo-only version to `~/.claude/settings.json`:
```json
{
  "hooks": {
    "PreCompact": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "echo 'COMPACTION IMMINENT. You MUST immediately: 1) Update MEMORY.md ACTIVE WORK. 2) Commit unsaved work. Do these NOW before context is lost.'"
          }
        ]
      }
    ]
  }
}
```
If the user wanted notifications (answer 4), read `notification-hooks.md` and set up the full notification stack (Telegram/SMS/desktop). Ask for credentials if needed.

---

## Step 3: Show Available Add-Ons

After core setup is done, show the user what else is available:

```
Core setup is done! Here's what's running:
- [list of files created]
- SessionStart hook checks project health every conversation
- PreCompact hook protects against context loss
- Auto-memory persists knowledge across sessions

**Optional features** — enable any of these anytime by asking me:

| Feature | Command | Best for |
|---------|---------|----------|
| **Research tracking** | "Set up research tracking" | Projects with iteration — dead ends, learnings, decisions |
| **Hours tracking** | "Set up hours tracking" | Track equivalent human work hours per session |
| **Multi-agent coordination** | "Set up multi-agent" | When 2+ Claude sessions work on the same project |
| **Notification hooks** | "Set up notifications" | Telegram/SMS alerts (if not done already) |
| **PostToolUse hooks** | "Set up post-edit hooks" | Auto type-check after .tsx edits, auto pycache clear, etc. |

Each feature detects your existing setup and merges in — nothing gets overwritten.
Just ask when you need one, even weeks from now.
```

---

## Step 4: Handle Add-On Requests

When the user asks for a feature later (or right now), read the corresponding guide and follow its Auto-Setup section:

| User says | Read this guide |
|-----------|----------------|
| "Set up research tracking" / "track experiments" / "banned techniques" | `research-tracking.md` |
| "Set up hours tracking" / "track time" | `hours-tracking.md` |
| "Set up multi-agent" / "two agents" / "agent coordination" | `multi-agent-coordination.md` |
| "Set up notifications" / "Telegram" / "notify me" | `notification-hooks.md` |
| "Set up post-edit hooks" / "auto type check" | `notification-hooks.md` (PostToolUse section) |

Each guide's Auto-Setup section handles: detect existing -> ask what's needed -> create files -> verify.

---

## Step 5: Verify & Commit

After any setup (core or add-on):

1. **Verify**: Check that hooks fire correctly, rules are loaded, settings.json is valid JSON
2. **Commit**: Stage and commit all new files with message `docs: Set up Claude Code project infrastructure`
3. **Summarize**: List every file created/modified and any manual steps remaining (e.g., "Create a Telegram bot via @BotFather")

---

## Important Notes for Claude

- **Don't overwhelm**: Core setup = 4 questions max. Everything else is opt-in.
- **Don't guess**: If you need info, ask. Don't invent project structure.
- **Don't overwrite**: Always merge into existing files. Ask before replacing.
- **Do create real files**: Don't just show templates — write them to disk.
- **Do make it re-runnable**: Step 0 detection means this works on first run AND on subsequent "add feature X" runs.
- **Platform matters**: Windows needs `sys.stdout.reconfigure(encoding="utf-8", errors="replace")` in every Python script.
- **Keep CLAUDE.md lean**: Use `.claude/rules/` for detailed rules, CLAUDE.md for the index. Under 150 lines if possible.
- **Two levels of hooks**: User-level (`~/.claude/settings.json`) for global behavior. Project-level (`.claude/settings.json`) for project-specific automation.
