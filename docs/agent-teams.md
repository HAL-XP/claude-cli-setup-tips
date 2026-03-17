# Agent Teams (Experimental)

Agent teams coordinate multiple Claude Code instances working together. One session acts as the team lead, spawning teammates that work independently and communicate directly with each other.

## Status: Experimental

Agent teams are disabled by default. Enable with:

```json
{
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"
  }
}
```

Or in your environment: `export CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`

## How Agent Teams Differ from Subagents

| | Subagents | Agent Teams |
|---|---|---|
| **Context** | Own window, results return to caller | Own window, fully independent |
| **Communication** | Report back to main agent only | Teammates message each other directly |
| **Coordination** | Main agent manages all work | Shared task list, self-coordination |
| **Best for** | Focused tasks where only the result matters | Complex work requiring discussion |
| **Token cost** | Lower (results summarized) | Higher (each teammate is separate instance) |

## When Agent Teams Work Well

- **Research and review**: Multiple perspectives on a problem simultaneously
- **Debugging with competing hypotheses**: Test different theories in parallel
- **New independent modules**: Each teammate owns a separate piece
- **Cross-layer coordination**: Frontend + backend + tests, each owned by a different teammate

## When Agent Teams Do NOT Work

- **Sequential tasks**: One step depends on the previous
- **Same-file edits**: Teammates will overwrite each other
- **Simple implementation**: A single session is faster
- **Tight dependencies**: Too much coordination overhead

## Starting a Team

```
Create an agent team with 3 teammates:
- One focused on security review
- One on performance analysis
- One validating test coverage
```

Claude creates the team, spawns teammates, and coordinates work.

## Display Modes

- **In-process** (default): All teammates in your terminal. Use `Shift+Down` to cycle through them.
- **Split panes**: Each teammate gets its own pane. Requires tmux or iTerm2.

```json
{
  "teammateMode": "in-process"
}
```

## Interacting with Teammates

- `Shift+Down`: Cycle through teammates (in-process mode)
- Click into pane (split-pane mode)
- `Ctrl+T`: Toggle task list
- `Ctrl+B`: Background a running task

## Task Management

The shared task list coordinates work. Tasks have three states: pending, in progress, completed. Tasks can depend on other tasks.

The lead can assign tasks explicitly, or teammates self-claim available work.

## Known Limitations

- **No session resumption** for in-process teammates (after `/resume`)
- **Task status can lag**: Teammates may fail to mark tasks complete
- **Shutdown can be slow**: Teammates finish current work first
- **One team per session**: Clean up before starting a new one
- **No nested teams**: Teammates cannot spawn their own teams
- **Lead is fixed**: Cannot promote a teammate to lead
- **Split panes**: Not supported in VS Code terminal, Windows Terminal, or Ghostty

## Our Assessment

Based on production experience:

**Agent teams are better for research and review than for implementation.**

For implementation work, the orchestrator + subagent pattern (see [Orchestrator Pattern](orchestrator-pattern.md)) gives you more control:
- Explicit task descriptions per agent
- Verification gates between steps
- Clear ownership boundaries
- No coordination overhead between teammates

Agent teams shine when you want multiple perspectives on a problem where the "right answer" is not obvious, such as debugging a mysterious issue with competing hypotheses, or reviewing a design from security/performance/usability angles simultaneously.

## Best Practices

- **Start with 3-5 teammates** -- diminishing returns beyond that
- **5-6 tasks per teammate** keeps everyone productive
- **Give detailed spawn prompts** -- teammates do not inherit the lead's conversation history
- **Monitor and steer** -- don't let a team run unattended too long
- **Require plan approval** for risky tasks: teammates plan first, lead approves
