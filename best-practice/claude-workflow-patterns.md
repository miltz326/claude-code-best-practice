# Claude Code Workflow Patterns

![Last Updated](https://img.shields.io/badge/Last_Updated-Apr%2008%2C%202026-white?style=flat&labelColor=555) ![Version](https://img.shields.io/badge/Claude_Code-v2.1.92-blue?style=flat&labelColor=555)

Proven workflow patterns for getting the most out of Claude Code — from single-session techniques to multi-agent parallelism.

<table width="100%">
<tr>
<td><a href="../">← Back to Claude Code Best Practice</a></td>
<td align="right"><img src="../!/claude-jumping.svg" alt="Claude" width="60" /></td>
</tr>
</table>

---

## 1. Explore → Plan → Implement → Commit

The recommended four-phase workflow. Separating research from coding avoids solving the wrong problem.

| Phase | Mode | What Claude does |
|-------|------|-----------------|
| **Explore** | Plan Mode | Reads files, answers questions — no changes made |
| **Plan** | Plan Mode | Creates a detailed implementation plan |
| **Implement** | Normal Mode | Codes against the plan, verifies with tests |
| **Commit** | Normal Mode | Commits with a descriptive message, opens PR |

### Example prompts per phase

**Explore**
```
read /src/auth and understand how we handle sessions and login.
also look at how we manage environment variables for secrets.
```

**Plan**
```
I want to add Google OAuth. What files need to change?
What's the session flow? Create a plan.
```
Press `Ctrl+G` to open the plan in your text editor for direct editing before Claude proceeds.

**Implement**
```
implement the OAuth flow from your plan. write tests for the
callback handler, run the test suite and fix any failures.
```

**Commit**
```
commit with a descriptive message and open a PR
```

> **When to skip planning**: if you can describe the diff in one sentence, skip the plan. Planning is most useful when the change modifies multiple files or you're unfamiliar with the code.

---

## 2. Writer / Reviewer Pattern

Use two separate sessions: one writes the code, a fresh context reviews it. The reviewer is unbiased because it did not write the code.

| Session A — Writer | Session B — Reviewer |
|--------------------|----------------------|
| `Implement a rate limiter for our API endpoints` | *(wait for A to finish)* |
| *(share the file path)* | `Review the rate limiter implementation in @src/middleware/rateLimiter.ts. Look for edge cases, race conditions, and consistency with our existing middleware patterns.` |
| `Here's the review feedback: [paste]. Address these issues.` | |

**Variations:**
- **Test-first**: Session A writes tests, Session B writes code to pass them.
- **Security review**: Session A implements, Session B reviews with a security lens only.
- **Multi-reviewer**: Fan out to 3 reviewers (security / performance / test coverage) in parallel.

---

## 3. CLAUDE.md — Do / Don't

A good CLAUDE.md is the highest-leverage single configuration change. Keep it concise and actionable.

| ✅ Include | ❌ Exclude |
|-----------|-----------|
| Bash commands Claude cannot guess | Anything Claude can figure out by reading code |
| Code style rules that differ from defaults | Standard language conventions Claude already knows |
| Testing instructions and preferred test runners | Detailed API documentation (link to docs instead) |
| Repository etiquette (branch naming, PR conventions) | Information that changes frequently |
| Architectural decisions specific to your project | Long explanations or tutorials |
| Developer environment quirks (required env vars) | File-by-file descriptions of the codebase |
| Common gotchas or non-obvious behaviors | Self-evident practices like "write clean code" |

**Target under 200 lines per file.** Bloated CLAUDE.md files cause Claude to ignore your actual instructions.

**Diagnostic tests:**
- If Claude keeps doing something wrong despite a rule against it → the file is too long and the rule is getting lost.
- If Claude asks questions answered in CLAUDE.md → the phrasing is ambiguous.
- For each line ask: *"Would removing this cause Claude to make mistakes?"* If not, cut it.

**Importing other files** — use `@path/to/import` syntax:
```markdown
See @README.md for project overview and @package.json for available npm commands.

# Additional Instructions
- Git workflow: @docs/git-instructions.md
- Personal overrides: @~/.claude/my-project-instructions.md
```
Imported files are expanded into context at launch. Maximum import depth is 5 hops.

---

## 4. Context Window Management

Context is the most important resource in Claude Code. LLM performance degrades as context fills.

### Track usage
- Add a [custom status line](/en/statusline) to watch context usage continuously.
- Use `/compact` manually at ~50% context to preserve important information proactively.
- For targeted compaction: `Esc+Esc` or `/rewind` → select a message checkpoint → **Summarize from here**.

### Best practices

| Situation | Action |
|-----------|--------|
| Switching to an unrelated task | `/clear` to reset context entirely |
| Same task, context getting noisy | `/compact Focus on the API changes` |
| Exploring a large codebase | Use subagents so exploration doesn't consume main context |
| Quick one-off question | `/btw` — answer appears in a dismissible overlay, never enters history |
| Multi-session task | `/rename` sessions with descriptive names; use `claude --continue` or `--resume` |

### Customize compaction behavior
Add instructions to CLAUDE.md:
```markdown
When compacting, always preserve the full list of modified files and any test commands.
```

---

## 5. Common Failure Patterns

| Pattern | Symptom | Fix |
|---------|---------|-----|
| **Kitchen sink session** | One session covers multiple unrelated tasks; context full of noise | `/clear` between tasks |
| **Correction loop** | Corrected Claude twice, still wrong | `/clear` and write a better initial prompt incorporating what you learned |
| **Bloated CLAUDE.md** | Claude ignores half the instructions | Ruthlessly prune; convert redundant rules to hooks for enforcement |
| **Trust-then-verify gap** | Implementation looks right but doesn't handle edge cases | Always provide verification (tests, scripts, screenshots) |
| **Infinite exploration** | Claude reads hundreds of files, fills context | Scope investigations narrowly or delegate to subagents |

---

## 6. Providing Verification Criteria

Claude performs dramatically better when it can check its own work.

| Strategy | Weak prompt | Strong prompt |
|----------|-------------|---------------|
| **Unit tests** | *"implement a function that validates email addresses"* | *"write a validateEmail function. test cases: user@example.com=true, invalid=false. run the tests after implementing"* |
| **Visual UI** | *"make the dashboard look better"* | *"[paste screenshot] implement this design. take a screenshot, compare to the original, list differences and fix them"* |
| **Root cause** | *"the build is failing"* | *"the build fails with this error: [paste]. fix it and verify the build succeeds. address the root cause, don't suppress the error"* |

---

## 7. Non-Interactive (Headless) Mode

Integrate Claude into CI pipelines, pre-commit hooks, or scripts with `claude -p`:

```bash
# One-off query
claude -p "Explain what this project does"

# Structured output for scripts
claude -p "List all API endpoints" --output-format json

# Pipe data directly
cat error.log | claude -p "Summarize the errors"

# Fan out across files
for file in $(cat files.txt); do
  claude -p "Migrate $file from React to Vue. Return OK or FAIL." \
    --allowedTools "Edit,Bash(git commit *)"
done
```

For uninterrupted CI runs, combine with `--permission-mode auto`:
```bash
claude --permission-mode auto -p "fix all lint errors"
```

---

## Sources

- [Best Practices — Claude Code Docs](https://code.claude.com/docs/en/best-practices)
- [Common Workflows — Claude Code Docs](https://code.claude.com/docs/en/common-workflows)
- [Memory — Claude Code Docs](https://code.claude.com/docs/en/memory)
- [Permission Modes — Claude Code Docs](https://code.claude.com/docs/en/permission-modes)
