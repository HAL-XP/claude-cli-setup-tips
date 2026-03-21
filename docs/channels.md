# Channels

Channels are MCP servers that push messages INTO a Claude Code session from external platforms. Two-way channels also let Claude reply back. Shipped in v2.1.80 as a research preview.

## How Channels Differ from Hooks

| | Hooks | Channels |
|---|-------|----------|
| **Direction** | One-way (Claude -> you) | Two-way (you <-> Claude) |
| **Trigger** | Event-driven (idle, permission, compaction) | Persistent connection, you send anytime |
| **Auth** | Works with API key auth | Requires claude.ai login |
| **Use case** | Automated system alerts | Interactive remote control |

You can use BOTH. Hooks for automated events, channels for human interaction.

## Telegram Channel (Official)

### Prerequisites

1. A Telegram bot (from BotFather) -- see [Telegram Notifications](telegram-notifications.md) for setup
2. Credentials stored in `~/.claude_credentials` (TELEGRAM_BOT_TOKEN, TELEGRAM_CHAT_ID)
3. Claude Code v2.1.80+
4. claude.ai login (not API key auth)

### Launch

```bash
claude --channels plugin:telegram@claude-plugins-official
```

### Pairing Flow

1. DM your bot on Telegram
2. The bot replies with a pairing code
3. Approve the pairing code in the CLI
4. Done -- two-way connection established

### What Claude Gets

Once paired, Claude receives tools for Telegram interaction:

- **Reply** -- send messages back to your Telegram chat
- **React** -- add reactions to messages
- **Edit** -- edit previously sent messages

You can send commands FROM Telegram and Claude will act on them in the active session.

## Discord Channel

Same concept, different platform:

```bash
claude --channels plugin:discord@claude-plugins-official
```

Requires a Discord server invite step. Otherwise the same pairing flow as Telegram.

## When to Keep Hooks vs Switch to Channels

**Keep hooks if:**
- You only need one-way alerts (permission prompts, compaction warnings)
- You use API key auth (channels require claude.ai login)
- You want automated system events without interaction

**Switch to channels if:**
- You want to send commands from your phone
- You need two-way communication with the session
- You want to monitor AND control remotely

**Use both (recommended):**
- Hooks for automated system events (PreCompact, permission prompts, smart relay)
- Channel for interactive control (sending commands, asking for status)

## Custom Channels

Build your own MCP channel server for webhooks, CI alerts, monitoring, or anything that needs to push messages into a Claude session.

### One-Way Channel

Declare the `claude/channel` capability and emit notifications:

```typescript
import { Server } from '@modelcontextprotocol/sdk/server/index.js'
import { StdioServerTransport } from '@modelcontextprotocol/sdk/server/stdio.js'

const mcp = new Server(
  { name: 'webhook', version: '0.0.1' },
  {
    capabilities: { experimental: { 'claude/channel': {} } },
    instructions: 'Events arrive as <channel source="webhook" ...>. Read and act.',
  },
)
await mcp.connect(new StdioServerTransport())

Bun.serve({
  port: 8788,
  hostname: '127.0.0.1',
  async fetch(req) {
    const body = await req.text()
    await mcp.notification({
      method: 'notifications/claude/channel',
      params: { content: body, meta: { path: new URL(req.url).pathname } },
    })
    return new Response('ok')
  },
})
```

This creates a local HTTP server that forwards any incoming webhook into the Claude session. Example: point your CI to `http://localhost:8788/ci-failure` and Claude gets notified of build failures in real time.

### Two-Way Channel

Add a reply tool to let Claude respond back through the channel:

```typescript
mcp.setRequestHandler(ListToolsRequestSchema, async () => ({
  tools: [{
    name: 'reply',
    description: 'Send a reply back through the channel',
    inputSchema: {
      type: 'object',
      properties: { message: { type: 'string' } },
      required: ['message'],
    },
  }],
}))
```

### Security: Gate Inbound Messages

Without sender gating, channels are a prompt injection vector. Always validate who is sending messages:

- Maintain a sender allowlist
- Verify tokens or signatures on incoming webhooks
- Do not expose the channel HTTP port to the public internet

### Running Custom Channels

During the research preview, custom channels require a special flag:

```bash
claude --dangerously-load-development-channels
```

Runtime requirements: Bun, Node, or Deno with `@modelcontextprotocol/sdk`.

## Requirements and Limitations

| Requirement | Detail |
|-------------|--------|
| **Auth** | claude.ai login only (not Console/API key) |
| **Version** | v2.1.80+ |
| **Team/Enterprise** | Must explicitly enable channels in org settings |
| **Preview status** | Research preview -- approved allowlist is Anthropic-curated |
| **Custom channels** | Need `--dangerously-load-development-channels` flag |

## Lessons Learned

**What works:**
- Telegram channel for phone-based monitoring and quick commands
- Pairing flow is simple and secure (one-time code approval)
- Combining hooks (system alerts) + channel (human interaction) -- best of both worlds
- Custom webhook channel for CI failure notifications

**What does NOT work:**
- Channels without sender gating -- prompt injection vector
- Expecting channels to replace ALL hooks -- hooks are better for automated system events
- Using channels with API key auth -- requires claude.ai login only
- Running custom channels without the `--dangerously-load-development-channels` flag during preview
