# Rules Directory (`.claude/rules/`)

Rules are auto-loaded markdown files that give Claude domain-specific instructions. They keep your CLAUDE.md lean while ensuring Claude always has the right context for each domain.

## How It Works

Files in `.claude/rules/` are loaded into every conversation for the project. Claude sees them without being asked. You can also have user-level rules at `~/.claude/rules/` that apply to all projects.

```
.claude/rules/
  frontend.md          # React/Tailwind/shadcn patterns
  python-api.md        # FastAPI routes, restart protocol
  pipeline.md          # Stage order, naming conventions
  ux.md                # UX principles, state-driven UI
  hours-tracking.md    # Hours log format
  banned-techniques.md # Dead ends -- never retry these
  pipeline-knowledge.md # Cross-cutting pipeline knowledge
```

## Path-Scoped Rules

Rules can be scoped to specific file types using YAML frontmatter:

```markdown
---
paths:
  - "src/api/**/*.ts"
---

# API Development Rules

- All API endpoints must include input validation
- Use the standard error response format
- Include OpenAPI documentation comments
```

These rules only load when Claude is working with files matching the glob pattern. This saves context space and reduces noise.

**Glob pattern examples:**

| Pattern | Matches |
|---------|---------|
| `**/*.ts` | All TypeScript files |
| `src/**/*` | All files under `src/` |
| `*.md` | Markdown files in project root |
| `src/components/*.tsx` | React components in specific dir |
| `src/**/*.{ts,tsx}` | TS and TSX files under src |

Rules without a `paths` field are loaded unconditionally.

## Benefits Over a Large CLAUDE.md

- **Each file stays focused** -- easy to find and update rules for one domain
- **CLAUDE.md stays lean** -- under 200 lines, just the index
- **Rules are always loaded** -- no need to tell Claude to read them
- **Git-friendly** -- changes to one domain don't show up in CLAUDE.md diffs
- **Path-scoping** -- frontend rules only load when editing frontend files

## Real Example: `python-api.md`

```markdown
# API Rules (FastAPI Backend)

## Server
- Backend runs on `localhost:8000` with CORS enabled.
- MUST run with `.venv/Scripts/python.exe` (some deps only in venv).
- Restart command: `.venv/Scripts/python.exe -m uvicorn api.main:app --host 0.0.0.0 --port 8000 --reload`

## MANDATORY: Restart API After Backend Changes
After ANY change to Python files in `api/`, `pipeline/`, or `scripts/`:
1. Find uvicorn PIDs: `tasklist | grep python`
2. Kill them all: `taskkill //PID <pid> //F`
3. Clear pycache
4. Restart (in background)
5. Verify: `curl -s http://localhost:8000/health`

Do this AFTER committing, as the final step. Never skip it.

## Route Patterns
- Static routes MUST come before parameterized catch-all routes
- Heavy operations go through job queue -- never synchronous
- Return JSON for data, raise `HTTPException` for errors
```

## Real Example: `frontend.md`

```markdown
# Frontend Rules (React + Tailwind + shadcn/ui)

## Styling
- Use Tailwind utility classes exclusively. No CSS files, no inline `style={}`.
- Use `cn()` from `@/lib/utils` for conditional/merged classes.
- Use shadcn/ui semantic color tokens (bg-card, text-foreground, etc.)
- Dark theme is the only theme.
- Text sizes: only text-sm, text-xs, text-[0.65rem]. No other sub-sm sizes.

## Component Patterns
- All components receive `api: string` prop for the backend URL.
- Use `useActivity()` hook for async operation tracking.
- Fetch errors: catch, extract message, show via toast. Never silently swallow.
```

## Real Example: `ux.md`

```markdown
# UX Principles

## 1. State-driven UI, never interaction-driven
- All visual indicators derive from backend data state, not user clicks.
- useState for transient UI only (expanded/collapsed, hover).

## 2. Single source of truth
- Pipeline state flows from App.tsx via props.
- After mutations, trigger state refresh via onStateChange callback.

## 3. Validate before expensive actions
- Operations calling external APIs validate prerequisites first.
- Show inline warnings. Never fail silently.
- Always offer "proceed anyway" escape hatch.

## 4. Disabled means explained
- Every disabled button has a tooltip explaining WHY.
- Pattern: title={!ready ? "Generate X first" : "What this does"}

## 5. Custom dialogs, never browser-native
- NEVER use window.alert(), window.confirm(), window.prompt().
- Use the custom ConfirmDialog component via createPortal to document.body.
```

## The Banned Techniques File

The single most time-saving pattern for iterative projects. Without it, you will retry failed approaches across sessions.

```markdown
# Banned Techniques (proven harmful or dead -- do NOT retry)

## Libraries
- Library X -- memory leak in production (tested 2026-03-05)
- Library Y -- no TypeScript types, poor DX (evaluated 2026-03-06)

## Approaches
- LPIPS for image similarity detection -- requires GPU, unreliable results (3 attempts)
- Grid screenshots for per-item QA -- items too small to analyze, need per-item captures

## Dead Ends
- Blanket rotation fix for 3D assets -- fixes some items, breaks others
- Agent Teams for implementation work -- too much coordination overhead
```

**Why `.claude/rules/`:** Files here are auto-loaded into every session. Claude sees them without being asked and literally cannot forget.

**Key rules:**
- Add entries as soon as something is confirmed dead
- Include the date and context
- Group by category
- Differentiate "harmful" (makes worse) from "dead" (doesn't help)

## Cross-Project Rules with Symlinks

The `.claude/rules/` directory supports symlinks:

```bash
ln -s ~/shared-claude-rules .claude/rules/shared
ln -s ~/company-standards/security.md .claude/rules/security.md
```

Circular symlinks are detected and handled gracefully.

## User-Level Rules

Personal rules at `~/.claude/rules/` apply to every project on your machine. Use for preferences that are not project-specific. User-level rules load before project rules (project rules have higher priority).
