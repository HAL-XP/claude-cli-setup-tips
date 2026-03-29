# Session Guardian (External Daemon)

A Python daemon that runs independently of Claude and monitors session reliability. It auto-relaunches crashed sessions, validates credentials, maintains lock files, and sends notifications -- all without depending on the session it is protecting.

## The Problem

Claude Code sessions can die for several reasons:
- API errors or rate limits
- Network interruption
- System sleep/wake cycle (especially on laptops)
- Memory pressure (system kills the process)
- Accidental terminal closure

When a session dies:
- In-progress work may be uncommitted
- Telegram channel connection is lost (no remote access)
- The user does not know until they check the terminal
- Manual relaunch is required, with correct flags and credentials

If you are relying on Claude for monitoring, CI alerts, or remote access via Telegram, a dead session means you are blind until you notice.

## Architecture

```
session-guardian.py (daemon, runs in its own terminal)
  |
  +-- Monitors: Claude process PID (polling every 30s)
  +-- Writes: session.lock (prevents app from killing session)
  +-- Validates: Telegram token in .env matches instance config
  +-- On crash: Auto-relaunches with correct flags
  +-- Notifies: Telegram on crash, relaunch, or failure
```

The guardian runs in a separate terminal window. It does NOT run inside Claude's session -- that would defeat the purpose, since the guardian needs to survive when the session dies.

## The Script

```python
#!/usr/bin/env python3
"""
Session Guardian -- external daemon for Claude session reliability.

Usage:
    python session-guardian.py --project-dir /path/to/project [--interval 30]
"""

import argparse
import json
import os
import re
import subprocess
import time
from datetime import datetime, timezone
from pathlib import Path

HOME = Path(os.environ.get("USERPROFILE", os.environ.get("HOME", "~")))
CREDENTIALS = HOME / ".claude_credentials"
CHECK_INTERVAL = 30   # seconds between checks
MAX_RESTARTS = 10     # max restarts before giving up
RESTART_COOLDOWN = 5  # seconds between restart attempts


def load_credentials() -> dict:
    """Load credentials from ~/.claude_credentials."""
    creds = {}
    if not CREDENTIALS.exists():
        return creds
    for line in CREDENTIALS.read_text().splitlines():
        line = line.strip()
        if not line or line.startswith("#"):
            continue
        if line.startswith("export "):
            line = line[7:]
        if "=" in line:
            key, val = line.split("=", 1)
            creds[key.strip()] = val.strip().strip("'\"")
    return creds


def find_session(session_name: str) -> dict | None:
    """Find a Claude process matching this session name."""
    try:
        # Windows: use PowerShell CIM query
        result = subprocess.run(
            ["powershell", "-NoProfile", "-Command",
             'Get-CimInstance Win32_Process -Filter '
             '"Name=\'claude.exe\'" | '
             'Select-Object ProcessId,CommandLine | '
             'ConvertTo-Csv -NoTypeInformation'],
            capture_output=True, text=True, timeout=10
        )
        for line in result.stdout.strip().split("\n")[1:]:
            match = re.match(r'"(\d+)","(.+)"', line)
            if not match:
                continue
            pid = int(match.group(1))
            cmd = match.group(2).replace('""', '"')
            if f'-n "{session_name}"' in cmd or f"-n {session_name}" in cmd:
                return {
                    "pid": pid,
                    "cmd": cmd,
                    "has_channels": "--channels" in cmd,
                }
    except Exception as e:
        log(f"Process scan error: {e}")
    return None


def write_lock(lock_path: Path, pid: int, has_channels: bool, name: str):
    """Write session lock file."""
    lock_data = {
        "pid": pid,
        "hasChannels": has_channels,
        "instanceName": name,
        "startedAt": datetime.now(timezone.utc).isoformat(),
        "guardianPid": os.getpid(),
    }
    lock_path.parent.mkdir(parents=True, exist_ok=True)
    lock_path.write_text(json.dumps(lock_data, indent=2))


def send_notification(creds: dict, message: str):
    """Send a Telegram notification."""
    token = creds.get("TELEGRAM_BOT_TOKEN", "")
    chat_id = creds.get("TELEGRAM_CHAT_ID", "")
    if not token or not chat_id:
        return
    try:
        subprocess.run(
            ["curl", "-s",
             f"https://api.telegram.org/bot{token}/sendMessage",
             "-d", f"chat_id={chat_id}",
             "--data-urlencode", f"text=[Guardian] {message}"],
            capture_output=True, timeout=10
        )
    except Exception:
        pass


def relaunch(project_dir: Path, session_name: str) -> int | None:
    """Relaunch Claude session."""
    # Option 1: Use a launch script if it exists
    launch_script = project_dir / "_scripts" / "launch.sh"
    if launch_script.exists():
        cmd = ["bash", str(launch_script)]
    else:
        # Option 2: Direct launch with standard flags
        cmd = [
            "claude",
            "--dangerously-skip-permissions",
            "--continue",
            "-n", session_name,
            "--channels", "plugin:telegram@claude-plugins-official",
            "--permission-mode", "bypassPermissions",
        ]

    try:
        # Launch in a new terminal window (detached from guardian)
        if os.name == "nt":
            subprocess.Popen(
                ["cmd", "/c", "start", "cmd", "/k"] + cmd,
                cwd=str(project_dir),
            )
        else:
            subprocess.Popen(
                ["nohup"] + cmd,
                cwd=str(project_dir),
                stdout=open("/tmp/guardian-relaunch.log", "w"),
                stderr=subprocess.STDOUT,
            )

        # Wait for process to appear
        time.sleep(20)
        proc = find_session(session_name)
        return proc["pid"] if proc else None
    except Exception as e:
        log(f"Relaunch error: {e}")
        return None


def log(msg: str):
    ts = datetime.now().strftime("%H:%M:%S")
    print(f"[{ts}] {msg}", flush=True)


def main():
    parser = argparse.ArgumentParser(description="Session Guardian")
    parser.add_argument("--project-dir", required=True, help="Project directory")
    parser.add_argument("--session-name", default="Claude", help="Session name (-n flag)")
    parser.add_argument("--interval", type=int, default=CHECK_INTERVAL)
    parser.add_argument("--lock-path", default=None, help="Lock file path")
    args = parser.parse_args()

    project_dir = Path(args.project_dir)
    lock_path = Path(args.lock_path) if args.lock_path else project_dir / ".claude" / "session.lock"
    creds = load_credentials()
    restart_count = 0

    log(f"=== Session Guardian started ===")
    log(f"Project: {project_dir}")
    log(f"Session: {args.session_name}")
    log(f"Lock: {lock_path}")
    log(f"Interval: {args.interval}s")

    send_notification(creds, f"Guardian online. Monitoring '{args.session_name}'.")

    while True:
        try:
            proc = find_session(args.session_name)

            if proc:
                # Session alive -- update lock, reset counter
                write_lock(lock_path, proc["pid"], proc["has_channels"], args.session_name)
                restart_count = 0

                if not proc["has_channels"]:
                    log(f"WARNING: Session PID {proc['pid']} has no --channels flag")
            else:
                # Session dead
                log("SESSION DEAD!")

                if restart_count >= MAX_RESTARTS:
                    log(f"Max restarts ({MAX_RESTARTS}) reached. Giving up.")
                    send_notification(creds, f"CRITICAL: Session crashed {MAX_RESTARTS} times. Manual intervention needed.")
                    break

                restart_count += 1
                send_notification(creds, f"Session crashed. Auto-relaunching ({restart_count}/{MAX_RESTARTS})...")

                time.sleep(RESTART_COOLDOWN)
                new_pid = relaunch(project_dir, args.session_name)

                if new_pid:
                    write_lock(lock_path, new_pid, True, args.session_name)
                    send_notification(creds, f"Session relaunched (PID {new_pid}).")
                else:
                    send_notification(creds, f"Relaunch failed. Retrying in {args.interval}s...")

        except KeyboardInterrupt:
            log("Guardian stopped.")
            send_notification(creds, "Guardian stopped.")
            break
        except Exception as e:
            log(f"Error: {e}")

        time.sleep(args.interval)

    if lock_path.exists():
        lock_path.unlink()
    log("=== Guardian exited ===")


if __name__ == "__main__":
    main()
```

## Usage

Open a separate terminal and run:

```bash
python session-guardian.py \
  --project-dir /path/to/your/project \
  --session-name "MyProject" \
  --interval 30
```

The guardian will:
1. Find the Claude process matching your session name
2. Write a `session.lock` file with the PID and metadata
3. Check every 30 seconds that the process is still alive
4. Auto-relaunch if the session dies (up to 10 times)
5. Send Telegram notifications on crash and recovery

## The Lock File

The `session.lock` file serves double duty:

**1. Session protection:** Other processes (like an Electron app's session lifecycle) can read the lock to know a Claude session is active and should not be killed.

```json
{
  "pid": 12345,
  "hasChannels": true,
  "instanceName": "MyProject",
  "startedAt": "2026-03-28T10:30:00Z",
  "guardianPid": 67890
}
```

**2. Crash detection:** If the PID in the lock file is no longer alive, the session died without cleanup.

## Adapting for macOS/Linux

Replace the PowerShell `Get-CimInstance` call with `pgrep`:

```python
def find_session(session_name: str) -> dict | None:
    try:
        result = subprocess.run(
            ["pgrep", "-af", f"claude.*-n.*{session_name}"],
            capture_output=True, text=True, timeout=5
        )
        for line in result.stdout.strip().split("\n"):
            if not line:
                continue
            pid = int(line.split()[0])
            cmd = line
            return {
                "pid": pid,
                "cmd": cmd,
                "has_channels": "--channels" in cmd,
            }
    except Exception:
        pass
    return None
```

## Running as a System Service

For always-on monitoring, run the guardian as a system service:

**systemd (Linux):**

```ini
[Unit]
Description=Claude Session Guardian
After=network.target

[Service]
Type=simple
ExecStart=/usr/bin/python3 /path/to/session-guardian.py --project-dir /path/to/project --session-name MyProject
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

**Windows Task Scheduler:**

Create a scheduled task that runs at logon:
- Program: `python`
- Arguments: `session-guardian.py --project-dir D:\Projects\my-project --session-name MyProject`
- Trigger: At logon
- Settings: "If the task fails, restart every 1 minute"

## Lessons Learned

**What works:**
- Running independently of Claude -- the guardian survives what it monitors
- Lock file for cross-process coordination -- prevents accidental session kills
- Telegram notifications on crash -- you know immediately, even from your phone
- Capped restart count (10) -- prevents infinite restart loops on persistent errors

**What does NOT work:**
- Running the guardian inside the Claude session -- it dies when the session dies
- Polling too frequently (< 10s) -- creates CPU load for no benefit
- Relaunching without `--continue` -- starts a fresh session instead of resuming
- No cooldown between restarts -- rapid restart loops can cause system instability
