# Example CLAUDE.md (from a production project)

This is a simplified version of a real CLAUDE.md from a production multi-agent project. Details have been generalized.

```markdown
# Content Pipeline

Automation system that processes structured input data through multiple
AI-powered stages, generating 3D assets and assembling them into
interactive scenes.

## Agent Contexts

| Agent | Context File | Scope |
|---|---|---|
| **Pipeline** | `@docs/AGENT_PIPELINE.md` | Data parsing, LLM extraction, processing stages |
| **Frontend** | `@docs/AGENT_FRONTEND.md` | FastAPI backend, React UI, job system, UX design |
| **Builder** | `@docs/AGENT_BUILDER.md` | 3D engine integration, scene assembly, asset import |
| **Assets** | `@docs/AGENT_ASSETS.md` | Marketplace sourcing, mesh generation, import pipeline |

The shared contract between all agents is JSON: `docs/DATA_SCHEMA.md`

## Stack

- **Python**: FastAPI backend (`localhost:8000`), pipeline stages, engine bridge
- **React**: Vite + TypeScript frontend (`localhost:3000`)
- **CSS**: Tailwind CSS v4 + shadcn/ui
- **LLM**: Claude API for structured data extraction
- **Image gen**: AI image generation for reference images
- **3D mesh**: Local GPU generation primary, API fallback
- **Job queue**: Redis (local) for async GPU jobs

## Key Conventions

- All pipeline stages consume/produce JSON per `docs/DATA_SCHEMA.md`
- Entity slugs: `TYPE. DESCRIPTIVE NAME - VARIANT` -> `type_descriptive_name_variant`
- Entity IDs: `ENT_001_NAME`, `ENT_002_NAME` (stable across stages)
- LLM calls logged to `logs/llm_calls/`
- GPU jobs go through job queue -- never synchronous in API routes
- Never import engine-specific modules outside `pipeline/engine_bridge/`
- API keys in `~/.claude_credentials` (bash-sourceable), loaded as env vars

## Engineering Principles

### Think Systemically, Not Per-Instance
- Never fix a single instance of a problem. Fix the root cause.
- Every bug is a signal that a system is missing or broken.

### Design for Scale
- Input may have 5 items or 500.
- Every UI must handle 0, 1, few, and many items gracefully.
- Batch operations must support partial failure and progress tracking.

### Autonomous Verification
- After implementing a feature, verify it works.
- Use Playwright MCP to screenshot and visually inspect UI changes.
- Hit API endpoints with curl to verify responses.
- If you cannot verify, say so explicitly.

## Session Start Protocol

When you see `SessionStart` hook output containing an `ACTION:` line,
execute those steps immediately before responding to the user:
1. Read `MEMORY.md` for current state
2. Commit any uncommitted work
3. Restart API if not running

Do not wait for the user to ask. Do not ask permission. Just do it.

## What NOT To Do

- Do not commit large binary assets or credentials to git
- Do not modify `docs/DATA_SCHEMA.md` without updating all validators
- Do not call heavy APIs synchronously in API routes
```

## Key Takeaways

- **Under 100 lines** -- detailed rules live in `.claude/rules/`
- **Agent table** with `@path` imports for per-agent context
- **Engineering principles** that apply to ALL work
- **Session Start Protocol** paired with SessionStart hook
- **What NOT To Do** as explicit guardrails
- No detailed API routes, no component lists, no implementation details
