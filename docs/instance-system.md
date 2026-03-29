# Multi-Instance System

Run multiple isolated copies of the same project (or different projects) with separate data directories, API ports, credentials, and Telegram bots. Each instance is a full clone of the repository with its own identity, so they never conflict.

## The Problem

You want to run multiple Claude-assisted environments simultaneously:
- A main development instance
- A work assistant for a different project
- A personal assistant with different settings
- A second language variant with its own TTS voice

Without isolation, they fight over:
- Data directories (settings, favorites, history)
- API ports (both try to bind the same port)
- Telegram bot tokens (messages go to the wrong session)
- Process management (killing one kills the other)

## Architecture

```
~/.my-app/                          # Main instance data
~/.my-app/instances/work/           # Clone: work assistant
~/.my-app/instances/personal/       # Clone: personal assistant

D:/Projects/my-app/                 # Main repo (no instance.json)
D:/Projects/work-assistant/         # Clone repo (has instance.json)
D:/Projects/personal-assistant/     # Clone repo (has instance.json)
```

The main instance has no `instance.json` -- it uses the base data directory. Each clone has an `instance.json` that routes its data to an isolated subdirectory.

## Setup

### 1. Clone the Repository

```bash
git clone https://github.com/your/project.git work-assistant
cd work-assistant
```

### 2. Create instance.json

```json
{
  "id": "work-assistant",
  "name": "Work Assistant",
  "port": 19410,
  "description": "Work-focused AI assistant"
}
```

Key rules:
- `id` must be unique across all instances (used for directory naming)
- `port` must be unique per instance (main: 19400, work: 19410, personal: 19420)
- `name` is the human-readable label shown in UI and notifications

### 3. Install and Build

```bash
npm install
# Run any native compilation steps
npm run build
```

### 4. Configure Credentials

Each clone can use a different bot token for Telegram. Store all tokens in `~/.claude_credentials`:

```bash
# Main instance
export TELEGRAM_BOT_TOKEN="111:AAA..."

# Clone instances (use TELEGRAM_MAIN_BOT_TOKEN or per-instance keys)
export TELEGRAM_MAIN_BOT_TOKEN="222:BBB..."
export TELEGRAM_WORK_ASSISTANT_BOT_TOKEN="333:CCC..."
```

The session health check script automatically selects the right token based on the instance name.

## The Resolution Module

The core of the system is a single module that every other module imports:

```typescript
// src/main/instance.ts
import { readFileSync, existsSync, mkdirSync } from 'fs'
import { join } from 'path'

interface InstanceConfig {
  id: string
  name: string
  port?: number
  description?: string
}

const HOME = process.env.USERPROFILE || process.env.HOME || ''
const BASE_DIR = join(HOME, '.my-app')
const INSTANCE_JSON = join(process.cwd(), 'instance.json')

let _config: InstanceConfig | null = null
let _loaded = false

function loadConfig(): InstanceConfig | null {
  if (_loaded) return _config
  _loaded = true
  try {
    if (existsSync(INSTANCE_JSON)) {
      const raw = JSON.parse(readFileSync(INSTANCE_JSON, 'utf-8'))
      if (raw.id && typeof raw.id === 'string') {
        _config = raw as InstanceConfig
      }
    }
  } catch { /* no instance.json = main mode */ }
  return _config
}

/** True when running as a clone */
export function isClone(): boolean {
  return loadConfig() !== null
}

/** Per-instance data directory (auto-creates) */
export function getDataDir(): string {
  const config = loadConfig()
  const dir = config
    ? join(BASE_DIR, 'instances', config.id)
    : BASE_DIR
  if (!existsSync(dir)) mkdirSync(dir, { recursive: true })
  return dir
}

/** Resolve a path inside the instance data dir */
export function dataPath(...segments: string[]): string {
  return join(getDataDir(), ...segments)
}

/** HTTP API port */
export function getPort(): number {
  return loadConfig()?.port ?? 19400
}

/** Instance name */
export function getInstanceName(): string {
  return loadConfig()?.name ?? 'My App'
}
```

Every module that reads or writes data uses `dataPath()` instead of hardcoded paths:

```typescript
// Before (breaks with multiple instances):
const settingsFile = join(HOME, '.my-app', 'settings.json')

// After (instance-aware):
import { dataPath } from './instance'
const settingsFile = dataPath('settings.json')
```

## Token Routing

The session health check (run as a `SessionStart` hook) validates that the correct Telegram bot token is active:

```bash
#!/usr/bin/env bash
# Detect instance
INSTANCE_FILE="$CLAUDE_PROJECT_DIR/instance.json"
IS_CLONE="false"
TOKEN_KEY="TELEGRAM_BOT_TOKEN"

if [ -f "$INSTANCE_FILE" ]; then
    IS_CLONE="true"
    INSTANCE_NAME=$(grep '"name"' "$INSTANCE_FILE" | sed 's/.*: *"\([^"]*\)".*/\1/')
    # Try instance-specific token first, then shared clone token
    UPPER_NAME=$(echo "$INSTANCE_NAME" | tr '[:lower:]' '[:upper:]' | tr ' ' '_')
    TOKEN_KEY="TELEGRAM_${UPPER_NAME}_BOT_TOKEN"
    EXPECTED="${!TOKEN_KEY:-}"
    if [ -z "$EXPECTED" ]; then
        TOKEN_KEY="TELEGRAM_MAIN_BOT_TOKEN"
        EXPECTED="${TELEGRAM_MAIN_BOT_TOKEN:-}"
    fi
else
    EXPECTED="${TELEGRAM_BOT_TOKEN:-}"
fi

# Validate .env matches expected token
ENV_FILE="$HOME/.claude/channels/telegram/.env"
if [ -f "$ENV_FILE" ]; then
    ACTUAL=$(grep '^TELEGRAM_BOT_TOKEN=' "$ENV_FILE" | cut -d'=' -f2-)
    if [ "$ACTUAL" != "$EXPECTED" ]; then
        echo "TOKEN MISMATCH: .env has wrong token for this instance"
        echo "Fixing: writing $TOKEN_KEY to .env"
        echo "TELEGRAM_BOT_TOKEN=$EXPECTED" > "$ENV_FILE"
    fi
fi
```

## Port Allocation

Reserve a port range and assign per instance:

| Instance | Port | HTTPS Port |
|----------|------|-----------|
| Main | 19400 | 19401 |
| Work Assistant | 19410 | 19411 |
| Personal | 19420 | 19421 |
| Test/Dev | 19430 | 19431 |

Use 10-port gaps to leave room for additional services per instance.

## Launch Scripts

Each clone gets its own launch script that sets up the correct environment:

```bash
#!/usr/bin/env bash
# _scripts/launch-clone.sh

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
REPO_DIR="$(cd "$SCRIPT_DIR/.." && pwd)"

# Load credentials
source "$HOME/.claude_credentials" 2>/dev/null

# Detect instance and set correct bot token
if [ -f "$REPO_DIR/instance.json" ]; then
    INSTANCE_NAME=$(grep '"name"' "$REPO_DIR/instance.json" | sed 's/.*: *"\([^"]*\)".*/\1/')
    # Write correct token to .env for Telegram channel plugin
    UPPER=$(echo "$INSTANCE_NAME" | tr '[:lower:]' '[:upper:]' | tr ' ' '_')
    TOKEN_VAR="TELEGRAM_${UPPER}_BOT_TOKEN"
    TOKEN="${!TOKEN_VAR:-$TELEGRAM_MAIN_BOT_TOKEN}"
    mkdir -p "$HOME/.claude/channels/telegram"
    echo "TELEGRAM_BOT_TOKEN=$TOKEN" > "$HOME/.claude/channels/telegram/.env"
fi

cd "$REPO_DIR"
claude --dangerously-skip-permissions \
  --continue \
  -n "$INSTANCE_NAME" \
  --channels plugin:telegram@claude-plugins-official \
  --permission-mode bypassPermissions
```

## .gitignore for Clones

Main repos and clones need slightly different `.gitignore` files. Create a `.gitignore.clone` template:

```gitignore
# Clone-specific ignores
instance.json
node_modules/
dist/
out/
```

When setting up a clone: `cp .gitignore.clone .gitignore`

## Process Isolation

Each instance tracks its own PIDs:

```
~/.my-app/session.lock              # Main instance lock
~/.my-app/instances/work/session.lock   # Work clone lock
```

The lock file contains the process PID and metadata, ensuring that restarting one instance does not kill another. See [Process Management](tips-and-tricks.md#process-management-pid-tracking) for the PID tracking pattern.

## Lessons Learned

**What works:**
- Single `instance.ts` module that everything imports -- one source of truth for isolation
- Token routing via `SessionStart` hook -- catches mismatches before they cause problems
- 10-port gaps between instances -- leaves room for future services
- Lock files per instance -- prevents cross-instance process kills

**What does NOT work:**
- Hardcoded paths anywhere in the codebase -- every path must go through `dataPath()`
- Sharing a single Telegram bot between instances -- messages get routed to the wrong session
- Sharing a data directory -- settings from one instance overwrite another
- Forgetting to copy `.gitignore.clone` -- the clone commits instance-specific files
