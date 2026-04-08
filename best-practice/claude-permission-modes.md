# Permission Modes Best Practice

![Last Updated](https://img.shields.io/badge/Last_Updated-Apr%2008%2C%202026-white?style=flat&labelColor=555) ![Version](https://img.shields.io/badge/Claude_Code-v2.1.92-blue?style=flat&labelColor=555)

Six permission modes that control how often Claude asks before editing files or running commands.

<table width="100%">
<tr>
<td><a href="../">← Back to Claude Code Best Practice</a></td>
<td align="right"><img src="../!/claude-jumping.svg" alt="Claude" width="60" /></td>
</tr>
</table>

---

## Overview

| Mode | What runs without asking | Best for |
|------|--------------------------|----------|
| `default` | Reads only | Getting started, sensitive work |
| `acceptEdits` | Reads and file edits | Iterating on code you're reviewing |
| `plan` | Reads only | Exploring a codebase before changing it |
| `auto` | Everything, with background safety checks | Long tasks, reducing prompt fatigue |
| `dontAsk` | Only pre-approved tools | Locked-down CI and scripts |
| `bypassPermissions` | Everything except protected paths | Isolated containers and VMs only |

Regardless of mode, writes to **protected paths** are never auto-approved (`.git`, `.vscode`, `.claude`, shell RC files, `.gitconfig`, `.mcp.json`, etc.).

---

## 1. `default` Mode

Standard mode. Claude prompts before any file write, shell command, or MCP tool call. Only reads are auto-approved.

**Use when**: starting a new task, working with sensitive code, or onboarding to a codebase.

---

## 2. `acceptEdits` Mode

Auto-approves file creates and edits in your working directory. Writes to protected paths and all non-edit actions (shell commands, network) still prompt.

**Use when**: you want to review changes via `git diff` after the fact rather than approving each edit inline.

```bash
claude --permission-mode acceptEdits
```

Press `Shift+Tab` once from default mode to enter it.

---

## 3. `plan` Mode

Research-and-propose mode. Claude reads files, runs shell commands to explore, and writes a plan — but does not edit your source. Equivalent to `default` mode permissions, just constrained by Claude's behavior.

**Use when**: exploring a codebase before making changes, designing an implementation approach.

```bash
claude --permission-mode plan
# Or prefix a single prompt:
/plan <your question>
```

**When the plan is ready**, Claude presents it and asks how to proceed. Options:
- Approve and start in auto mode
- Approve and accept edits
- Approve and review each edit manually
- Keep planning with feedback

---

## 4. `auto` Mode (Research Preview)

> Requires Claude Code v2.1.83+. Available on Team/Enterprise plans with Claude Sonnet 4.6 or Opus 4.6 via Anthropic API only (not Bedrock/Vertex).

A separate classifier model reviews every action before it runs. Claude executes without permission prompts — the classifier blocks what looks risky.

**Use when**: long autonomous tasks, batch migrations, reducing prompt fatigue.

```bash
# Enable for the session and add to Shift+Tab cycle:
claude --enable-auto-mode

# Or start directly in auto mode:
claude --permission-mode auto -p "fix all lint errors"
```

### What the classifier blocks by default

**Blocked:**
- `curl | bash` patterns (download-and-execute)
- Sending sensitive data to external endpoints
- Production deploys and migrations
- Mass deletion on cloud storage
- Granting IAM or repo permissions
- Force push or direct push to `main`

**Allowed:**
- Local file operations in your working directory
- Installing dependencies from lock files
- Reading `.env` and sending credentials to their matching API
- Read-only HTTP requests
- Pushing to the branch you started on

Run `claude auto-mode defaults` to see the full built-in rule lists.

### Fallback behavior

Each blocked action shows a notification under the **Recently denied** tab in `/permissions` (press `r` to retry with manual approval).

Auto mode pauses and resumes prompting if:
- The classifier blocks **3 consecutive actions**
- The classifier blocks **20 total actions** in the session

In non-interactive mode (`-p`), repeated blocks abort the session entirely.

### Classifier architecture

The classifier sees user messages, tool calls, and your CLAUDE.md content. Tool results are stripped — hostile content in a file or web page cannot manipulate it directly. A separate server-side probe scans incoming tool results and flags suspicious content before Claude reads it.

---

## 5. `dontAsk` Mode

Auto-denies every tool that is not explicitly allowed by `permissions.allow` rules. Explicit `ask` rules are also denied rather than prompting. Fully non-interactive for CI pipelines.

```bash
claude --permission-mode dontAsk
```

**Use when**: pre-defining exactly what Claude may do in locked-down CI environments.

---

## 6. `bypassPermissions` Mode

Disables all permission prompts and safety checks. Tool calls execute immediately. Only writes to **protected paths** still prompt.

```bash
claude --permission-mode bypassPermissions
# Equivalent:
claude --dangerously-skip-permissions
```

**Use only in**: isolated environments (containers, VMs, devcontainers without internet). Offers no protection against prompt injection.

> Administrators can block this mode by setting `permissions.disableBypassPermissionsMode` to `"disable"` in managed settings.

---

## Switching Modes

### During a session

Press `Shift+Tab` to cycle through modes. Default cycle: `default` → `acceptEdits` → `plan`.

Optional modes slot in after `plan`:
- `bypassPermissions` appears after you start with `--permission-mode bypassPermissions` or `--allow-dangerously-skip-permissions`
- `auto` appears after you opt in with `--enable-auto-mode`

If both are enabled, the cycle is: `default` → `acceptEdits` → `plan` → `bypassPermissions` → `auto`.

`dontAsk` never appears in the cycle; set it only via the `--permission-mode` flag.

### At startup

```bash
claude --permission-mode plan
claude --permission-mode auto
claude --enable-auto-mode   # adds auto to cycle, does not activate immediately
```

### As a persistent default

Set `defaultMode` in `.claude/settings.json` or `~/.claude/settings.json`:

```json
{
  "permissions": {
    "defaultMode": "acceptEdits"
  }
}
```

Valid values: `default`, `acceptEdits`, `plan`, `auto`, `dontAsk`, `bypassPermissions`.

---

## Protected Paths

These paths are never auto-approved in any mode:

**Directories:** `.git`, `.vscode`, `.idea`, `.husky`, `.claude` (except `.claude/commands`, `.claude/agents`, `.claude/skills`, `.claude/worktrees`)

**Files:** `.gitconfig`, `.gitmodules`, `.bashrc`, `.bash_profile`, `.zshrc`, `.zprofile`, `.profile`, `.ripgreprc`, `.mcp.json`, `.claude.json`

---

## Sources

- [Permission Modes — Claude Code Docs](https://code.claude.com/docs/en/permission-modes)
- [Permissions — Claude Code Docs](https://code.claude.com/docs/en/permissions)
- [CLI Reference — Claude Code Docs](https://code.claude.com/docs/en/cli-reference)
