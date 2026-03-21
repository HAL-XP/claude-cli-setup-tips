# Scheduled Tasks

Scheduled tasks let Claude re-run prompts automatically on an interval. Use `/loop` for quick setup or cron tools for precise control. Available since v2.1.72.

## /loop Command

The quickest way to set up a recurring task. Type it inline with your prompt:

```
/loop 5m check if the deployment finished
```

### Interval Syntax

| Position | Example | Result |
|----------|---------|--------|
| Leading | `/loop 30m run the integration tests` | Every 30 minutes |
| Trailing | `check the build status every 2 hours` | Every 2 hours |
| Default | `/loop run type checks` | Every 10 minutes |

### Units

| Unit | Meaning | Notes |
|------|---------|-------|
| `s` | Seconds | Rounded up to 1 minute minimum |
| `m` | Minutes | Most common |
| `h` | Hours | Good for monitoring |
| `d` | Days | Rare, but works |

### Loop Over Other Commands

`/loop` can wrap any slash command:

```
/loop 20m /review-pr 1234
/loop 30m /simplify
/loop 1h /agents qa-verifier
```

## One-Time Reminders

Natural language works for one-shot reminders:

```
remind me at 3pm to push the release branch
in 45 minutes, check whether the integration tests passed
after lunch, run the full test suite
```

These fire once and disappear. All times are interpreted in your local timezone.

## Cron Tools

For precise scheduling, use the built-in cron tools directly:

### CronCreate

Standard 5-field cron expression: `minute hour day-of-month month day-of-week`

```
CronCreate: */5 * * * *    "check CI status and report"
CronCreate: 0 */2 * * *    "run type checks on the frontend"
CronCreate: 30 17 * * 1-5  "remind me to commit and push"
```

Standard cron syntax applies: `*/5` (every 5), `1,3,5` (specific values), `1-5` (ranges), `0-59/15` (steps).

### CronList

Lists all active scheduled tasks with their IDs:

```
CronList
```

### CronDelete

Cancel a task by its ID (from CronList output):

```
CronDelete: <task-id>
```

### Limits

- Maximum 50 concurrent tasks per session
- 3-day auto-expiry for recurring tasks
- 1-minute minimum granularity

## Use Cases for Development

### Deployment Monitoring

```
/loop 5m check if the deploy finished and notify me
```

Set and forget. Claude checks, reports when done.

### Devlog Reminders

```
/loop 2h log hours and write a summary of what was done
```

Keeps the `_devlog/` up to date during long sessions.

### Code Quality

```
/loop 30m /simplify
/loop 20m run tsc --noEmit and report any new errors
```

Recurring quality gates that catch drift.

### CI/PR Babysitting

```
/loop 5m check the CI status for PR #42
/loop 10m are there any new review comments on PR #42
```

### End-of-Day Reminder

```
remind me at 5:30pm to commit, push, and update MEMORY.md
```

## How It Works

- Tasks fire **between your turns**, never mid-response
- If Claude is busy when a task fires, it is **queued until idle**
- Small jitter is added to avoid API thundering herd on short intervals
- All times use your **local timezone**
- Tasks run in the current session context -- they can see your project files, use tools, etc.

## Limitations

**Session-scoped.** All scheduled tasks live in memory. They are gone the moment you exit Claude Code.

| Limitation | Detail |
|------------|--------|
| No persistence | Tasks do not survive restart |
| 3-day expiry | Recurring tasks auto-expire after 3 days |
| No catch-up | If a fire was missed (machine asleep), it is skipped |
| 1-minute floor | Intervals below 1 minute are rounded up |
| Queued when busy | Tasks wait if Claude is mid-response -- they do not interrupt |

For anything that must survive restarts, use Desktop scheduled tasks or GitHub Actions with a `schedule` trigger.

## Desktop Scheduled Tasks (Durable)

The Claude Code Desktop app supports **persistent** scheduled tasks that survive restarts:

- Tasks run as long as the app is open
- Configured through the Desktop UI (visual schedule builder)
- Primary way to automate recurring work durably
- Not available in the CLI -- Desktop only

Use Desktop scheduled tasks when reliability matters more than convenience.

## Disable Scheduling

Kill all scheduling features with one environment variable:

```bash
export CLAUDE_CODE_DISABLE_CRON=1
```

This disables `/loop`, cron tools, and one-time reminders entirely.

## Lessons Learned

**What works:**
- `/loop` for deployment monitoring -- set and forget, Claude reports when done
- Devlog reminders with `/loop 2h` -- ensures hours tracking actually happens during long sessions
- One-shot reminders for end-of-day commits -- prevents the "forgot to push" problem
- Combining `/loop` with custom skills (`/loop 20m /review-pr 1234`) -- recurring checks with specialized logic
- Short polling intervals (5m) for CI status -- fast feedback without manual checking

**What does NOT work:**
- Relying on session-scoped tasks for critical automation -- if the session dies, the task dies. Use GitHub Actions for anything important
- Very short intervals (<1m) -- cron has 1-minute granularity, and sub-minute polling wastes API calls
- Expecting tasks to fire during long tool executions -- they queue, so a 30-minute build means your 5m loop fires late
- Forgetting about 3-day expiry -- long-running sessions need task recreation
- Scheduling tasks that produce verbose output on every fire -- the noise drowns out real information
