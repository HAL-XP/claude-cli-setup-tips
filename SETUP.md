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
- .claude/skills/*/SKILL.md — existing skills?
- ~/.claude/CLAUDE.md — global user instructions?
- ~/.claude/settings.json — existing hooks? Which types?
- ~/.claude_credentials or .env — existing credentials file?
- */MEMORY.md in auto-memory directory — existing memory?
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
- [x] ~/.claude/settings.json — hooks: PreCompact, Notification
- [x] etc.

**Not yet set up:**
- [ ] No MEMORY.md (session continuity)
- [ ] No /resume skill
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

### B. MEMORY.md (auto-memory directory)
Create initial file per `session-continuity.md` Pattern 1.

### C. `.claude/skills/resume/SKILL.md`
Create per `session-continuity.md` Pattern 2. Start simple — can be extended later.

### D. `.claude/rules/banned-techniques.md`
Create empty template. The user will populate it as dead ends are discovered.
```markdown
# Banned Techniques (proven harmful or dead — do NOT retry)
## [Add categories as you discover dead ends]
(none yet)
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
        "type": "command",
        "command": "echo 'COMPACTION IMMINENT. You MUST immediately: 1) Update MEMORY.md ACTIVE WORK. 2) Commit unsaved work. Do these NOW before context is lost.'"
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
- Start every session with `/resume`, end by telling me to save state

**Optional features** — enable any of these anytime by asking me:

| Feature | Command | Best for |
|---------|---------|----------|
| **Research tracking** | "Set up research tracking" | Projects with iteration — dead ends, learnings, decisions |
| **Hours tracking** | "Set up hours tracking" | Track equivalent human work hours per session |
| **Multi-agent coordination** | "Set up multi-agent" | When 2+ Claude sessions work on the same project |
| **Cloud GPU support** | "Set up cloud GPU" | ML training on Vast.ai, RunPod, etc. |
| **Notification hooks** | "Set up notifications" | Telegram/SMS alerts (if not done already) |

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
| "Set up cloud GPU" / "Vast.ai" / "RunPod" | Ask: which provider, API keys, GPU types. Add section to CLAUDE.md. |

Each guide's Auto-Setup section handles: detect existing → ask what's needed → create files → verify.

---

## Step 5: Verify & Commit

After any setup (core or add-on):

1. **Verify**: Run `/resume` to confirm it reads MEMORY.md correctly
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
