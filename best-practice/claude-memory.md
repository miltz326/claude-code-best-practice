# Claude Memory

![Last Updated](https://img.shields.io/badge/Last_Updated-Apr%2008%2C%202026-white?style=flat&labelColor=555) ![Version](https://img.shields.io/badge/Claude_Code-v2.1.92-blue?style=flat&labelColor=555)

Persistent context via CLAUDE.md files — how to write them and how they load in monorepos.

<table width="100%">
<tr>
<td><a href="../">← Back to Claude Code Best Practice</a></td>
<td align="right"><img src="../!/claude-jumping.svg" alt="Claude" width="60" /></td>
</tr>
</table>

---

## 1. Writing a Good CLAUDE.md

A well-structured CLAUDE.md is the single most impactful way to improve Claude Code's output for your project. Humanlayer has an excellent guide covering what to include, how to structure it, and common pitfalls.

- [Humanlayer - Writing a good Claude.md](https://www.humanlayer.dev/blog/writing-a-good-claude-md)

---

## 2. CLAUDE.md in Large Monorepos

When working with Claude Code in a monorepo, understanding how CLAUDE.md files are loaded into context is crucial for organizing your project instructions effectively.

<p align="center">
  <a href="https://x.com/bcherny/status/2016339448863355206"><img src="assets/claude-memory/claude-memory-monorepo.jpg" alt="CLAUDE.md loading in monorepos" width="600"></a>
</p>

### The Two Loading Mechanisms

Claude Code uses two distinct mechanisms for loading CLAUDE.md files:

#### Ancestor Loading (UP the directory tree)

When you start Claude Code, it walks **upward** from your current working directory toward the filesystem root and loads every CLAUDE.md it finds along the way. These files are loaded **immediately at startup**.

#### Descendant Loading (DOWN the directory tree)

CLAUDE.md files in subdirectories below your current working directory are **NOT loaded at launch**. They are only included when Claude reads files in those subdirectories during your session. This is known as **lazy loading**.

### Example Monorepo Structure

Consider a typical monorepo with separate directories for different components:

```
/mymonorepo/
├── CLAUDE.md          # Root-level instructions (shared across all components)
├── frontend/
│   └── CLAUDE.md      # Frontend-specific instructions
├── backend/
│   └── CLAUDE.md      # Backend-specific instructions
└── api/
    └── CLAUDE.md      # API-specific instructions
```

### Scenario 1: Running Claude Code from the Root Directory

When you run Claude Code from `/mymonorepo/`:

```bash
cd /mymonorepo
claude
```

| File | Loaded at Launch? | Reason |
|------|-------------------|--------|
| `/mymonorepo/CLAUDE.md` | Yes | It's your current working directory |
| `/mymonorepo/frontend/CLAUDE.md` | No | Loaded only when you read/edit files in `frontend/` |
| `/mymonorepo/backend/CLAUDE.md` | No | Loaded only when you read/edit files in `backend/` |
| `/mymonorepo/api/CLAUDE.md` | No | Loaded only when you read/edit files in `api/` |

### Scenario 2: Running Claude Code from a Component Directory

When you run Claude Code from `/mymonorepo/frontend/`:

```bash
cd /mymonorepo/frontend
claude
```

| File | Loaded at Launch? | Reason |
|------|-------------------|--------|
| `/mymonorepo/CLAUDE.md` | Yes | It's an ancestor directory |
| `/mymonorepo/frontend/CLAUDE.md` | Yes | It's your current working directory |
| `/mymonorepo/backend/CLAUDE.md` | No | Different branch of the directory tree |
| `/mymonorepo/api/CLAUDE.md` | No | Different branch of the directory tree |

### Key Takeaways

1. **Ancestors always load at startup** — Claude walks UP the directory tree and loads all CLAUDE.md files it finds. This ensures you always have access to root-level, repository-wide instructions.

2. **Descendants load lazily** — Subdirectory CLAUDE.md files only load when you interact with files in those subdirectories. This prevents irrelevant context from bloating your session.

3. **Siblings never load** — If you're working in `frontend/`, you won't get `backend/CLAUDE.md` or `api/CLAUDE.md` loaded into context.

4. **Global CLAUDE.md** — You can also place a CLAUDE.md at `~/.claude/CLAUDE.md` in your home folder, which applies to ALL Claude Code sessions regardless of project.

### Why This Design Works for Monorepos

- **Shared instructions propagate down** — Root-level CLAUDE.md contains repository-wide conventions, coding standards, and common patterns that apply everywhere.

- **Component-specific instructions stay isolated** — Frontend developers don't need backend-specific instructions cluttering their context, and vice versa.

- **Context is optimized** — By lazily loading descendant CLAUDE.md files, Claude Code avoids loading potentially hundreds of kilobytes of irrelevant instructions at startup.

### Best Practices

1. **Put shared conventions in root CLAUDE.md** — Coding standards, commit message formats, PR templates, and other repository-wide guidelines.

2. **Put component-specific instructions in component CLAUDE.md** — Framework-specific patterns, component architecture, testing conventions unique to that component.

3. **Use CLAUDE.local.md for personal preferences** — Add it to `.gitignore` for instructions that shouldn't be shared with the team.

---

## 3. Importing Files into CLAUDE.md

CLAUDE.md files can import additional files using `@path/to/import` syntax. Imported files are expanded and loaded into context at launch alongside the CLAUDE.md that references them.

```markdown
See @README.md for project overview and @package.json for available npm commands.

# Additional Instructions
- Git workflow: @docs/git-instructions.md
- Personal overrides: @~/.claude/my-project-instructions.md
```

**Rules:**
- Both relative and absolute paths are supported
- Relative paths resolve relative to the file containing the import, not the working directory
- Imported files can recursively import other files — maximum depth of 5 hops
- The first time Claude Code encounters external imports in a project, it shows an approval dialog; if declined, imports stay disabled

**Sharing instructions across git worktrees:** A gitignored `CLAUDE.local.md` only exists in the worktree where you created it. To share personal instructions across all worktrees of the same repo, import from your home directory:

```markdown
# Individual Preferences
- @~/.claude/my-project-instructions.md
```

**Importing an `AGENTS.md`** used by other coding agents:

```markdown
@AGENTS.md

## Claude Code
Use plan mode for changes under `src/billing/`.
```

---

## 4. What to Write in CLAUDE.md (Do / Don't)

A bloated CLAUDE.md causes Claude to ignore your actual instructions. Target **under 200 lines** per file and ruthlessly prune.

| ✅ Include | ❌ Exclude |
|-----------|-----------|
| Bash commands Claude cannot guess | Anything Claude can figure out by reading code |
| Code style rules that differ from defaults | Standard language conventions Claude already knows |
| Testing instructions and preferred test runners | Detailed API documentation (link to docs instead) |
| Repository etiquette (branch naming, PR conventions) | Information that changes frequently |
| Architectural decisions specific to your project | Long explanations or tutorials |
| Developer environment quirks (required env vars) | File-by-file descriptions of the codebase |
| Common gotchas or non-obvious behaviors | Self-evident practices like "write clean code" |

**Diagnostic tests:**
- If Claude keeps doing something wrong despite a rule against it → the file is too long and the rule is getting lost
- If Claude asks questions that are answered in CLAUDE.md → the phrasing is ambiguous
- For each line ask: *"Would removing this cause Claude to make mistakes?"* If not, cut it

**Emphasis improves adherence.** Adding `IMPORTANT` or `YOU MUST` to a rule makes Claude follow it more reliably.

**Large CLAUDE.md files:** Use `@path/to/import` imports or split instructions into `.claude/rules/` files. Rules in `.claude/rules/` can be scoped to specific file types via `paths` frontmatter:

```markdown
---
paths:
  - "src/api/**/*.ts"
---

# API Rules
- All endpoints must include input validation
- Use the standard error response format
```

---

## Sources

- [Claude Code Documentation - Memory](https://code.claude.com/docs/en/memory)
- [Boris Cherny on X - Clarification on CLAUDE.md Loading](https://x.com/bcherny/status/2016339448863355206)
- [Humanlayer - Writing a good Claude.md](https://www.humanlayer.dev/blog/writing-a-good-claude-md)
