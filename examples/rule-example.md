# Example Rule Files

Real rule files from a production project. These live in `.claude/rules/` and are auto-loaded into every conversation.

## Frontend Rules (`frontend.md`)

```markdown
# Frontend Rules (React + Tailwind + shadcn/ui)

## Styling

- Use Tailwind utility classes exclusively. No CSS files, no inline `style={}` (except for dynamic values like computed colors).
- Use `cn()` from `@/lib/utils` for conditional/merged classes:
  ```tsx
  className={cn("base-classes", condition && "conditional-classes")}
  ```
- Use shadcn/ui semantic color tokens (`bg-card`, `text-foreground`, `border-border`, `bg-primary`, etc.) instead of hardcoded hex values.
- Dark theme is the only theme. `class="dark"` is set on `<html>`.
- **Text size tiers**: Use only two sub-`text-sm` sizes -- `text-[0.65rem]` (micro: badges, timestamps) and `text-xs` (12px: secondary content). Never use other sub-sm sizes.

## Component Patterns

- All components receive `api: string` prop for the backend URL.
- Use `useActivity()` hook from `ActivityContext` to track async operations.
- Fetch errors: catch, extract message, show via toast. Never silently swallow.
- Background data polling: `useEffect` + `setInterval` with cleanup.

## Common Tailwind Patterns

- Card: `bg-card border border-border rounded-md overflow-hidden`
- Primary button: `bg-primary text-primary-foreground border-none px-3 py-1.5 rounded cursor-pointer text-sm disabled:opacity-50`
- Badge: `bg-secondary text-secondary-foreground px-2 py-0.5 rounded text-[0.65rem]`
- Placeholder: `text-muted-foreground/50 text-sm italic`

## Path Alias

- `@/*` maps to `./src/*`. Use `@/lib/utils`, `@/components/ui/button`, etc.
```

## API Rules (`python-api.md`)

```markdown
# API Rules (FastAPI Backend)

## Server
- Backend runs on `localhost:8000` with CORS enabled for the frontend dev server.
- MUST run with `.venv/Scripts/python.exe` (some deps only installed in venv).
- Restart command: `.venv/Scripts/python.exe -m uvicorn api.main:app --host 0.0.0.0 --port 8000 --reload`

## MANDATORY: Restart API After Backend Changes

After ANY change to Python files in `api/`, `pipeline/`, or `scripts/`:
1. Find uvicorn PIDs: `tasklist | grep python`
2. Kill them all: `taskkill //PID <pid> //F` for each
3. Clear pycache: `find . -name "__pycache__" -type d -exec rm -rf {} +`
4. Restart (in background)
5. Verify: `sleep 3 && curl -s http://localhost:8000/health`

Do this AFTER committing, as the final step. Never skip it.

## Route Patterns
- Static route paths MUST come before parameterized catch-all routes
- Heavy operations (mesh gen, texture gen, LLM calls) go through job queue
- Return JSON for data, raise `HTTPException` for errors

## Job System
- Jobs use `JobScope(type, id)` for entity-scoped execution
- Background execution via `asyncio.to_thread`
- SSE log streaming at `GET /jobs/{id}/logs`
```

## UX Rules (`ux.md`)

```markdown
# UX Principles

## 1. State-driven UI, never interaction-driven
- All visual indicators derive from backend data state, not user clicks.
- useState for transient UI only (expanded/collapsed, hover).

## 2. Single source of truth
- Pipeline state flows from App.tsx via props -- components never fetch independently.
- After mutations, trigger state refresh via onStateChange callback.

## 3. Validate before expensive actions
- Operations calling external APIs validate prerequisites first.
- Show inline warnings with specific missing items.
- Always offer a "proceed anyway" escape hatch.

## 4. Disabled means explained
- Every disabled button has a tooltip explaining WHY.

## 5. Custom dialogs, never browser-native
- NEVER use window.alert(), window.confirm(), window.prompt().
- Use the custom ConfirmDialog component via createPortal to document.body.
- NEVER render modals inside #root if they set inert on #root.

## 6. Confirm before regenerating
- When a generation button's stage is already complete, show ConfirmDialog.
- Use variant="destructive" on confirm button since regeneration is destructive.
```

## Pipeline Knowledge Sync (`pipeline-knowledge.md`)

A rule that enforces documentation sync:

```markdown
# Pipeline Knowledge Base Maintenance

## Rule: Keep PIPELINE_KNOWLEDGE in sync

The file `api/assist.py` contains a `PIPELINE_KNOWLEDGE` string constant used as
the system prompt for the pipeline assistant.

Whenever you change any of the following, you MUST update PIPELINE_KNOWLEDGE:

1. Pipeline stages -- adding, removing, renaming, or reordering
2. Stage prerequisites -- changing what a stage requires
3. Stage inputs/outputs -- changing which files a stage reads or writes
4. Output file paths -- renaming or moving any output JSON file
5. Data schema -- structural changes

## Where to find the source of truth

| Topic | Source File |
|---|---|
| Stage order | `scripts/run_pipeline.py` |
| Parse stage | `pipeline/parser/` |
| Analysis | `pipeline/analysis/` |
| Layout | `pipeline/generation/layout.py` |
| Engine build | `pipeline/engine_bridge/builder.py` |
```

## Key Takeaways

- **Each file covers one domain** -- frontend, backend, UX, pipeline
- **Rules are specific and actionable** ("Use `cn()` from `@/lib/utils`" not "merge classes properly")
- **Include code patterns** that Claude should follow
- **Cross-cutting rules** (like pipeline-knowledge.md) enforce documentation sync
- **Mandatory sections** use bold `MANDATORY` or `MUST` language
- **Anti-patterns** are explicit ("NEVER use window.alert()")
