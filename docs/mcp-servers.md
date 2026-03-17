# MCP Servers

The Model Context Protocol (MCP) connects Claude Code to external tools: browsers, databases, APIs, and more. MCP servers give Claude capabilities beyond its built-in tools.

## What Is MCP

MCP is an open standard for AI-tool integrations. Claude Code connects to MCP servers that expose tools, and Claude can use those tools during conversations. As of March 2026, there are 200+ community MCP servers available.

## Configuration

MCP servers are configured in `.mcp.json` at the project root:

```json
{
  "mcpServers": {
    "playwright": {
      "type": "stdio",
      "command": "npx",
      "args": ["@playwright/mcp@latest"]
    },
    "github": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_TOKEN": "${GITHUB_TOKEN}"
      }
    }
  }
}
```

Transport modes: `stdio` (most common), `sse`, `http`, `ws`.

## Playwright MCP (Most Useful)

Playwright MCP gives Claude the ability to control a real browser: navigate pages, click buttons, fill forms, take screenshots, and extract data.

### Setup

```json
{
  "mcpServers": {
    "playwright": {
      "type": "stdio",
      "command": "npx",
      "args": ["@playwright/mcp@latest"]
    }
  }
}
```

### What It Enables

- **Visual verification**: Screenshot your app after making changes
- **End-to-end testing**: Click through UI flows to verify they work
- **Form filling**: Test form submissions and validations
- **State inspection**: Check that data appears correctly in the UI

### Real Usage

In a production project, Playwright MCP is used for:

1. **QA Verifier agent**: Screenshots every page state (empty, loaded, error, loading)
2. **UX Analyst agent**: Walks through user flows, takes screenshots for audit reports
3. **Frontend Implementation agent**: Verifies that new features render correctly
4. **Visual QA agent**: Captures per-asset screenshots for quality analysis

Example from agent template:

```markdown
# Verification Protocol (MANDATORY)

2. **Visual verification**: Use Playwright MCP to:
   - Navigate to the affected page/tab
   - Take a screenshot of the changed UI
   - Verify the feature visually works (not just compiles)
3. **Interaction test**: If you changed interactive elements:
   - Click/interact with them via Playwright
   - Verify the expected behavior occurs
```

### Common Playwright Tools

| Tool | Purpose |
|------|---------|
| `browser_navigate` | Go to a URL |
| `browser_snapshot` | Get page accessibility snapshot |
| `browser_take_screenshot` | Capture screenshot |
| `browser_click` | Click an element |
| `browser_fill_form` | Fill form fields |
| `browser_press_key` | Press keyboard keys |
| `browser_evaluate` | Run JavaScript in the page |
| `browser_wait_for` | Wait for element/condition |

## GitHub MCP

Access GitHub issues, PRs, files, and search:

```json
{
  "mcpServers": {
    "github": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_TOKEN": "${GITHUB_TOKEN}"
      }
    }
  }
}
```

Note: Claude Code also has built-in `gh` CLI access via Bash. The MCP server provides richer tool integration but the CLI is often sufficient.

## Scoping MCP to Subagents

You can give an MCP server to a specific subagent without exposing it to the main conversation:

```yaml
---
name: browser-tester
description: Tests features in a real browser
mcpServers:
  - playwright:
      type: stdio
      command: npx
      args: ["-y", "@playwright/mcp@latest"]
---

Use the Playwright tools to navigate and verify UI features.
```

The MCP server starts when the subagent launches and disconnects when it finishes. The main conversation context is not burdened with the MCP tool descriptions.

## MCP Tool Hooks

You can hook into MCP tool usage with PreToolUse/PostToolUse:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "mcp__playwright__.*",
        "hooks": [
          {
            "type": "command",
            "command": "echo 'Playwright tool being used'"
          }
        ]
      }
    ]
  }
}
```

MCP tool names follow the pattern: `mcp__<server>__<tool>`.

## Popular MCP Servers

| Server | Purpose | Package |
|--------|---------|---------|
| **Playwright** | Browser automation | `@playwright/mcp` |
| **GitHub** | Issues, PRs, search | `@modelcontextprotocol/server-github` |
| **Brave Search** | Web search | `@anthropic-ai/mcp-server-brave-search` |
| **Filesystem** | File operations | `@modelcontextprotocol/server-filesystem` |
| **Memory** | Persistent knowledge graph | `@modelcontextprotocol/server-memory` |

## Lessons Learned

**What works:**
- Playwright for visual verification -- catches issues code review misses
- Scoping MCP to subagents -- keeps main context clean
- `.mcp.json` in project root -- checked into git, whole team uses it

**What does NOT work:**
- Too many MCP servers at once -- tool descriptions consume context
- Expecting perfect browser interaction -- some complex UIs need manual testing
- Using MCP for things Claude's built-in tools handle better (file reading, grep)
