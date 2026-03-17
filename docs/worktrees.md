# Worktree Isolation

Git worktrees give each agent its own copy of the repository, preventing file conflicts when multiple agents work in parallel.

## The Problem

Without isolation, two agents editing the same file will overwrite each other's changes. Even agents working on different files can cause issues with shared state (package-lock.json, __pycache__, etc.).

## How It Works

A git worktree is an additional working directory attached to the same repository. Each worktree has its own checkout of a branch and its own independent files, while sharing git history, objects, and config.

Claude Code has native support via the `--worktree` flag (or `-w`):

```bash
# Start Claude in its own worktree
claude --worktree

# With a name
claude --worktree my-feature

# Shorthand
claude -w
```

## Subagent Isolation

The most common use is in agent templates. Add `isolation: worktree` to the frontmatter:

```yaml
---
name: feature-builder
description: Implements features in isolation
tools: ["Read", "Write", "Edit", "Glob", "Grep", "Bash"]
isolation: worktree
---

You are a feature implementation agent. You work in an isolated
git worktree so your changes don't conflict with other agents.
```

When the orchestrator spawns this agent:
1. Claude creates a temporary worktree (a full copy of the repo)
2. The agent works in its own directory
3. When done, changes are merged back to the main branch
4. If the agent makes no changes, the worktree is automatically cleaned up

## Parallel Development Pattern

Three agents working simultaneously:

```
Main branch: orchestrator running here
  |
  +-- Worktree A: Frontend agent (editing frontend/src/)
  |
  +-- Worktree B: Backend agent (editing api/)
  |
  +-- Worktree C: Pipeline agent (editing pipeline/)
```

Each agent has its own branch, its own files, zero conflicts.

## Sparse Checkout for Large Repos

For monorepos, use `worktree.sparsePaths` to only checkout relevant directories:

```json
{
  "worktree": {
    "sparsePaths": ["src/", "tests/", "package.json"]
  }
}
```

This dramatically reduces worktree creation time for large repositories.

## WorktreeCreate and WorktreeRemove Hooks

You can hook into worktree lifecycle events:

```json
{
  "hooks": {
    "WorktreeCreate": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "echo 'Worktree created at: ' && pwd"
          }
        ]
      }
    ],
    "WorktreeRemove": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "echo 'Cleaning up worktree'"
          }
        ]
      }
    ]
  }
}
```

## Batch Operations

The built-in `/batch` command uses worktree isolation to run parallel codebase migrations:

```
/batch "Update all API handlers to use the new error format"
```

This spawns worktree-isolated agents to process different parts of the codebase in parallel.

## Manual Parallel Sessions

You can also run multiple Claude Code sessions manually:

```bash
# Terminal 1
claude -w frontend-work

# Terminal 2
claude -w backend-work

# Terminal 3 (main)
claude
```

Each session has its own worktree and branch.

## Current Limitations

- **No environment isolation**: Worktrees share the same node_modules, virtualenv, etc. If your project needs isolated environments per worktree, you need additional tooling.
- **Merge conflicts**: If two worktrees modify the same lines, you get merge conflicts when merging back. Keep agent ownership boundaries clear.
- **Disk space**: Each worktree is a near-full copy of the repo (git shares objects, but working files are duplicated).

## Lessons Learned

**What works:**
- `isolation: worktree` in agent frontmatter -- seamless, automatic
- Clear ownership boundaries per agent -- prevents merge conflicts
- 2-3 parallel worktrees -- good throughput without merge headaches
- Automatic cleanup of empty worktrees -- no orphaned directories

**What does NOT work:**
- 5+ parallel worktrees -- merge conflicts become likely
- Agents without clear file ownership -- they edit the same files
- Expecting full environment isolation -- worktrees share runtime environments
- Not defining ownership boundaries before spawning parallel agents
