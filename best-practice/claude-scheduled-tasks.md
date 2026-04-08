# Scheduled Tasks Best Practice

![Last Updated](https://img.shields.io/badge/Last_Updated-Apr%2008%2C%202026-white?style=flat&labelColor=555) ![Version](https://img.shields.io/badge/Claude_Code-v2.1.92-blue?style=flat&labelColor=555)

Three ways to schedule recurring work in Claude Code — from quick in-session polling to persistent cloud automation.

<table width="100%">
<tr>
<td><a href="../">← Back to Claude Code Best Practice</a></td>
<td align="right"><img src="../!/claude-jumping.svg" alt="Claude" width="60" /></td>
</tr>
</table>

---

## Comparison: Three Scheduling Options

|  | Cloud Scheduled Tasks | Desktop Scheduled Tasks | `/loop` (in-session) |
|--|----------------------|------------------------|----------------------|
| Runs on | Anthropic cloud | Your machine | Your machine |
| Requires machine on | No | Yes | Yes |
| Requires open session | No | No | Yes |
| Persistent across restarts | Yes | Yes | No (session-scoped) |
| Access to local files | No (fresh clone) | Yes | Yes |
| MCP servers | Connectors configured per task | Config files and connectors | Inherits from session |
| Permission prompts | No (runs autonomously) | Configurable per task | Inherits from session |
| Minimum interval | 1 hour | 1 minute | 1 minute |
| How to configure | `/schedule` CLI command | Desktop app | `/loop` in session |

**Choosing the right option:**
- **Cloud tasks**: work that should run reliably without your machine on
- **Desktop tasks**: work needing local file access, running on your own machine
- **`/loop`**: quick polling during an active session

---

## 1. `/loop` — In-Session Recurring Prompts

The `/loop` bundled skill schedules a recurring prompt within the current session. The job fires in the background while the session stays open.

```
/loop 5m check if the deployment finished and tell me what happened
```

Claude parses the interval, converts it to a cron expression, schedules the job, and confirms the cadence and job ID.

### Interval syntax

| Form | Example | Interval |
|------|---------|----------|
| Leading token | `/loop 30m check the build` | every 30 minutes |
| Trailing `every` clause | `/loop check the build every 2 hours` | every 2 hours |
| No interval | `/loop check the build` | defaults to every 10 minutes |

Supported units: `s` (seconds), `m` (minutes), `h` (hours), `d` (days). Seconds round up to the nearest minute.

### Loop over another command

```
/loop 20m /review-pr 1234
```

Each time the job fires, Claude runs `/review-pr 1234` as if you had typed it.

### One-time reminders

Describe what you want in natural language instead:
```
remind me at 3pm to push the release branch
in 45 minutes, check whether the integration tests passed
```

### Managing scheduled tasks

```
what scheduled tasks do I have?
cancel the deploy check job
```

Underlying tools used by Claude:

| Tool | Purpose |
|------|---------|
| `CronCreate` | Schedule a task. Accepts a 5-field cron expression, prompt, and recurrence flag |
| `CronList` | List all scheduled tasks with IDs, schedules, and prompts |
| `CronDelete` | Cancel a task by its 8-character ID |

Maximum 50 scheduled tasks per session.

### How `/loop` runs

- The scheduler checks every second and enqueues tasks at low priority
- A task fires between your turns — not while Claude is mid-response
- If Claude is busy when a task comes due, it waits until the current turn ends
- All times use your **local timezone**
- Tasks expire automatically after **7 days** (fires once more, then deletes itself)
- Closing the terminal cancels all tasks

### Jitter

To avoid every session hitting the API at the same moment:
- Recurring tasks fire up to 10% of their period late, capped at 15 minutes
- One-shot tasks scheduled for `:00` or `:30` fire up to 90 seconds early
- Pick a minute like `03` instead of `00` to avoid the one-shot jitter

### Cron expression reference

`CronCreate` accepts standard 5-field expressions: `minute hour day-of-month month day-of-week`

| Expression | Meaning |
|------------|---------|
| `*/5 * * * *` | Every 5 minutes |
| `0 * * * *` | Every hour on the hour |
| `0 9 * * *` | Every day at 9am local |
| `0 9 * * 1-5` | Weekdays at 9am local |
| `30 14 15 3 *` | March 15 at 2:30pm local |

### Disable the scheduler

Set `CLAUDE_CODE_DISABLE_CRON=1` in your environment to disable `/loop` and all cron tools entirely.

---

## 2. Desktop Scheduled Tasks

Run on your local machine on a cron schedule — persistent across session restarts, no open session required. Access to your local files and tools.

Configure via the Desktop app or the `/schedule` CLI command. See [Desktop Scheduled Tasks docs](https://code.claude.com/docs/en/desktop-scheduled-tasks) for full setup.

**Best for:** local automations like nightly builds, file sync, or local health checks.

---

## 3. Cloud Scheduled Tasks

Run on Anthropic-managed infrastructure — persistent, requires no local machine, uses a fresh repository clone for each run.

Configure via the `/schedule` command in Claude Code:
```
/schedule
```

**Best for:** autonomous work that should run reliably overnight or over weekends — CI-style checks, scheduled reports, automated dependency updates.

MCP connectivity uses connectors configured per task. Local file access is not available (tasks clone from git).

---

## Sources

- [Scheduled Tasks — Claude Code Docs](https://code.claude.com/docs/en/scheduled-tasks)
- [Desktop Scheduled Tasks — Claude Code Docs](https://code.claude.com/docs/en/desktop-scheduled-tasks)
- [Web Scheduled Tasks — Claude Code Docs](https://code.claude.com/docs/en/web-scheduled-tasks)
