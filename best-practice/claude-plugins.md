# Plugins Best Practice

![Last Updated](https://img.shields.io/badge/Last_Updated-Apr%2008%2C%202026-white?style=flat&labelColor=555) ![Version](https://img.shields.io/badge/Claude_Code-v2.1.92-blue?style=flat&labelColor=555)

Plugins bundle skills, agents, hooks, and MCP servers into a single installable unit — shareable across projects and teams via marketplaces.

<table width="100%">
<tr>
<td><a href="../">← Back to Claude Code Best Practice</a></td>
<td align="right"><img src="../!/claude-jumping.svg" alt="Claude" width="60" /></td>
</tr>
</table>

---

## Plugins vs Standalone Configuration

| Approach | Skill names | Best for |
|----------|-------------|----------|
| **Standalone** (`.claude/` directory) | `/hello` | Personal workflows, quick experiments, single-project customizations |
| **Plugins** (`.claude-plugin/plugin.json`) | `/plugin-name:hello` | Sharing with teams, distributing to community, versioned releases, reuse across projects |

Start with standalone configuration for quick iteration, then convert to a plugin when ready to share.

---

## Plugin Structure

```
my-plugin/
├── .claude-plugin/
│   └── plugin.json        # Manifest (required)
├── commands/              # Skills as Markdown files
├── agents/                # Custom agent definitions
├── skills/                # Agent Skills with SKILL.md files
├── hooks/
│   └── hooks.json         # Event handlers
├── .mcp.json              # MCP server configurations
├── .lsp.json              # LSP server configurations
├── bin/                   # Executables added to Bash PATH
└── settings.json          # Default settings when plugin is enabled
```

> **Common mistake**: Do NOT put `commands/`, `agents/`, `skills/`, or `hooks/` inside `.claude-plugin/`. Only `plugin.json` goes inside `.claude-plugin/`. All other directories must be at the plugin root.

---

## `plugin.json` Manifest

```json
{
  "name": "my-plugin",
  "description": "Description shown in the plugin manager",
  "version": "1.0.0",
  "author": {
    "name": "Your Name"
  },
  "homepage": "https://github.com/you/my-plugin",
  "repository": "https://github.com/you/my-plugin",
  "license": "MIT"
}
```

| Field | Required | Purpose |
|-------|----------|---------|
| `name` | Yes | Unique identifier and skill namespace. Skills are prefixed: `/my-plugin:hello` |
| `description` | Yes | Shown in the plugin manager when browsing or installing |
| `version` | Yes | Semantic versioning for releases |
| `author` | No | Attribution |
| `homepage` | No | Documentation URL |
| `repository` | No | Source repository URL |
| `license` | No | SPDX license identifier |

---

## Managing Plugins with `claude plugin`

```bash
# Browse and install from marketplace
/plugin install my-plugin@marketplace-name
/plugin install fakechat@claude-plugins-official

# List installed plugins
/plugin list

# Update a plugin
/plugin update my-plugin

# Remove a plugin
/plugin remove my-plugin

# Add a marketplace source
/plugin marketplace add anthropics/claude-plugins-official

# Update marketplace index
/plugin marketplace update claude-plugins-official
```

After installing, run `/reload-plugins` to activate without restarting.

---

## Testing Plugins Locally

Use `--plugin-dir` during development — no installation required:

```bash
claude --plugin-dir ./my-plugin

# Load multiple plugins at once:
claude --plugin-dir ./plugin-one --plugin-dir ./plugin-two
```

A `--plugin-dir` plugin takes precedence over an installed marketplace plugin of the same name. Run `/reload-plugins` to pick up changes without restarting.

Test each component:
- Skills: `/plugin-name:skill-name`
- Agents: `/agents`
- Hooks: trigger the relevant event and observe

---

## Adding Skills to a Plugin

Skills are in `skills/<name>/SKILL.md`. The folder name becomes the skill name, namespaced to the plugin:

```markdown
<!-- skills/code-review/SKILL.md -->
---
name: code-review
description: Reviews code for best practices and potential issues. Use when reviewing code, checking PRs, or analyzing code quality.
---

When reviewing code, check for:
1. Code organization and structure
2. Error handling
3. Security concerns
4. Test coverage
```

Skills are invoked as `/my-plugin:code-review` or auto-invoked by Claude when the task matches the description.

Arguments are accepted via `$ARGUMENTS`:
```markdown
Greet the user named "$ARGUMENTS" warmly.
```
Invoked as: `/my-plugin:hello Alex`

---

## Adding LSP Servers (Code Intelligence)

For languages without an official LSP plugin, add `.lsp.json` at the plugin root:

```json
{
  "go": {
    "command": "gopls",
    "args": ["serve"],
    "extensionToLanguage": {
      ".go": "go"
    }
  }
}
```

> For common languages (TypeScript, Python, Rust), install pre-built LSP plugins from the official marketplace instead of creating custom ones.

---

## Default Settings in a Plugin

Include `settings.json` at the plugin root to apply default configuration when the plugin is enabled. Currently only the `agent` key is supported:

```json
{
  "agent": "security-reviewer"
}
```

This activates `security-reviewer` (from the plugin's `agents/` directory) as the main thread agent when the plugin is enabled.

---

## Plugin Marketplace

Submit to the official Anthropic marketplace:
- **Claude.ai**: [claude.ai/settings/plugins/submit](https://claude.ai/settings/plugins/submit)
- **Console**: [platform.claude.com/plugins/submit](https://platform.claude.com/plugins/submit)

The official community plugin source: [anthropics/claude-plugins-official](https://github.com/anthropics/claude-plugins-official)

---

## Converting Standalone Configuration to a Plugin

```bash
# 1. Create plugin structure
mkdir -p my-plugin/.claude-plugin
cat > my-plugin/.claude-plugin/plugin.json << 'EOF'
{
  "name": "my-plugin",
  "description": "Migrated from standalone configuration",
  "version": "1.0.0"
}
EOF

# 2. Copy existing files
cp -r .claude/commands my-plugin/
cp -r .claude/agents my-plugin/   # if any
cp -r .claude/skills my-plugin/   # if any
```

For hooks, create `my-plugin/hooks/hooks.json` and copy the `hooks` object from your `.claude/settings.json`.

| Standalone (`.claude/`) | Plugin |
|-------------------------|--------|
| Single project only | Shareable via marketplaces |
| Files in `.claude/commands/` | Files in `plugin-name/commands/` |
| Hooks in `settings.json` | Hooks in `hooks/hooks.json` |
| Manual copy to share | Install with `/plugin install` |

---

## Sources

- [Plugins — Claude Code Docs](https://code.claude.com/docs/en/plugins)
- [Plugins Reference — Claude Code Docs](https://code.claude.com/docs/en/plugins-reference)
- [Discover and Install Plugins — Claude Code Docs](https://code.claude.com/docs/en/discover-plugins)
- [claude-plugins-official repository](https://github.com/anthropics/claude-plugins-official)
