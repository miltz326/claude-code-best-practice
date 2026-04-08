# Agent Teams Best Practice

![Last Updated](https://img.shields.io/badge/Last_Updated-Apr%2008%2C%202026-white?style=flat&labelColor=555) ![Version](https://img.shields.io/badge/Claude_Code-v2.1.92-blue?style=flat&labelColor=555)

Experimental feature for coordinating multiple Claude Code instances as a team — with shared tasks, inter-agent messaging, and a designated team lead.

<table width="100%">
<tr>
<td><a href="../">← Back to Claude Code Best Practice</a></td>
<td align="right"><img src="../!/claude-jumping.svg" alt="Claude" width="60" /></td>
</tr>
</table>

---

> **Experimental**: Agent teams are disabled by default. Enable with `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`. Requires Claude Code v2.1.32 or later.

---

## Agent Teams vs Subagents

Both parallelize work, but they coordinate differently:

| | Subagents | Agent Teams |
|--|-----------|-------------|
| **Context** | Own context window; results return to the caller | Own context window; fully independent |
| **Communication** | Report results back to the main agent only | Teammates message each other directly |
| **Coordination** | Main agent manages all work | Shared task list with self-coordination |
| **Best for** | Focused tasks where only the result matters | Complex work requiring discussion and collaboration |
| **Token cost** | Lower — results summarized back to main context | Higher — each teammate is a separate Claude instance |

**Use subagents** when you need quick, focused workers that report back.  
**Use agent teams** when teammates need to share findings, challenge each other, and coordinate on their own.

---

## Enable Agent Teams

```bash
# Via environment variable:
export CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1
claude

# Or persist in settings.json:
```
```json
{
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"
  }
}
```

---

## Starting a Team

Tell Claude what you need in natural language. Claude creates the team, spawns teammates, and coordinates work:

```
I'm designing a CLI tool that helps developers track TODO comments across
their codebase. Create an agent team to explore this from different angles:
one teammate on UX, one on technical architecture, one playing devil's advocate.
```

Claude creates a team with a shared task list, spawns teammates, and attempts cleanup when finished.

---

## Display Modes

| Mode | Description | Requirement |
|------|-------------|-------------|
| `auto` | Split panes if inside tmux, in-process otherwise | Default |
| `in-process` | All teammates run inside your main terminal; use `Shift+Down` to cycle | Any terminal |
| `tmux` | Each teammate gets its own pane — see all output at once | tmux or iTerm2 |

Set `teammateMode` in `~/.claude.json`:
```json
{
  "teammateMode": "in-process"
}
```

Or override per session:
```bash
claude --teammate-mode in-process
```

**Installing tmux for split-pane mode:**
- macOS: `brew install tmux` or via your system package manager
- Launch with iTerm2 using `tmux -CC` for best results
- Enable iTerm2 Python API: **iTerm2 → Settings → General → Magic → Enable Python API**

**In-process navigation:**
- `Shift+Down` — cycle through teammates
- `Ctrl+T` — toggle the task list
- `Esc` — interrupt a teammate's current turn

---

## Best Practices

### Team size

Start with **3–5 teammates** for most workflows. This balances parallel work with manageable coordination overhead.

- Token costs scale linearly with teammates
- Aim for **5–6 tasks per teammate** to keep everyone productive
- More teammates do not always mean faster results — coordination overhead increases

### Size tasks appropriately

- **Too small**: coordination overhead exceeds the benefit
- **Too large**: teammates work too long without check-ins
- **Just right**: self-contained units with a clear deliverable (a function, a test file, a review)

If the lead isn't creating enough tasks, ask it to split the work into smaller pieces.

### Avoid file conflicts

Two teammates editing the same file leads to overwrites. Break work so each teammate owns a different set of files.

### Give teammates enough context

Teammates do NOT inherit the lead's conversation history. Include task-specific details in the spawn prompt:

```
Spawn a security reviewer teammate with this prompt: "Review the authentication
module at src/auth/ for vulnerabilities. Focus on token handling, session
management, and input validation. The app uses JWT tokens in httpOnly cookies.
Report issues with severity ratings."
```

### Keep the lead waiting

The lead may start implementing itself instead of delegating. If you see this:
```
Wait for your teammates to complete their tasks before proceeding.
```

### Monitor and steer

Check in on teammates' progress regularly. Letting a team run unattended for too long risks wasted effort on incorrect approaches.

---

## Requiring Plan Approval

For complex or risky tasks, require teammates to plan before implementing:

```
Spawn an architect teammate to refactor the authentication module.
Require plan approval before they make any changes.
```

The teammate works in read-only plan mode until the lead approves. If rejected, the teammate revises and resubmits.

---

## Using Subagent Definitions as Teammates

Reference any subagent definition by name when spawning:

```
Spawn a teammate using the security-reviewer agent type to audit the auth module.
```

The teammate honors the definition's `tools` allowlist and `model`. Team coordination tools (`SendMessage`, task management) are always available even when `tools` restricts others.

> **Note**: `skills` and `mcpServers` frontmatter fields are not applied when a subagent definition runs as a teammate. Teammates load these from project and user settings.

---

## `TeammateIdle` Hook

Runs when a teammate is about to go idle. Exit with code 2 to send feedback and keep the teammate working:

```json
{
  "hooks": {
    "TeammateIdle": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "~/.claude/hooks/quality-gate.sh"
          }
        ]
      }
    ]
  }
}
```

Related hooks:
- `TaskCreated` — exit code 2 prevents task creation with feedback
- `TaskCompleted` — exit code 2 prevents task completion with feedback

---

## Team Architecture

| Component | Role |
|-----------|------|
| **Team lead** | The main Claude Code session that creates the team, spawns teammates, coordinates work |
| **Teammates** | Separate Claude Code instances working on assigned tasks |
| **Task list** | Shared list of work items with states: pending → in progress → completed |
| **Mailbox** | Messaging system for direct inter-agent communication |

Storage locations (auto-managed, do not edit by hand):
- Team config: `~/.claude/teams/{team-name}/config.json`
- Task list: `~/.claude/tasks/{team-name}/`

---

## Use Case Examples

### Parallel Code Review

```
Create an agent team to review PR #142. Spawn three reviewers:
- One focused on security implications
- One checking performance impact
- One validating test coverage
Have them each review and report findings.
```

### Competing Hypotheses for Debugging

```
Users report the app exits after one message instead of staying connected.
Spawn 5 agent teammates to investigate different hypotheses. Have them talk to
each other to try to disprove each other's theories, like a scientific debate.
Update the findings doc with whatever consensus emerges.
```

---

## Clean Up

When done, always ask the **lead** to clean up (not a teammate):

```
Clean up the team
```

Cleanup fails if active teammates are still running — shut them down first:
```
Ask the researcher teammate to shut down
```

---

## Known Limitations

- **No session resumption with in-process teammates**: `/resume` and `/rewind` do not restore in-process teammates
- **Task status can lag**: teammates sometimes fail to mark tasks as completed; update manually if stuck
- **Shutdown can be slow**: teammates finish their current request before shutting down
- **One team per session**: clean up before starting a new team
- **No nested teams**: only the lead can manage the team; teammates cannot spawn their own teams
- **Split panes not supported in**: VS Code integrated terminal, Windows Terminal, Ghostty
- **Permissions set at spawn**: all teammates start with the lead's permission mode

---

## Sources

- [Agent Teams — Claude Code Docs](https://code.claude.com/docs/en/agent-teams)
- [Sub-agents — Claude Code Docs](https://code.claude.com/docs/en/sub-agents)
- [Hooks — Claude Code Docs](https://code.claude.com/docs/en/hooks)
