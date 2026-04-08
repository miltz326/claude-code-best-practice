# GitHub Actions Integration Best Practice

![Last Updated](https://img.shields.io/badge/Last_Updated-Apr%2008%2C%202026-white?style=flat&labelColor=555) ![Version](https://img.shields.io/badge/Claude_Code-v2.1.92-blue?style=flat&labelColor=555)

Use `@claude` mentions in GitHub PRs and issues to trigger Claude Code automation — PR reviews, feature implementation, and bug fixes, all following your project's CLAUDE.md standards.

<table width="100%">
<tr>
<td><a href="../">← Back to Claude Code Best Practice</a></td>
<td align="right"><img src="../!/claude-jumping.svg" alt="Claude" width="60" /></td>
</tr>
</table>

---

## Quick Setup

The easiest setup is via Claude Code in your terminal:

```bash
/install-github-app
```

This guides you through installing the GitHub App and adding the `ANTHROPIC_API_KEY` secret. For manual setup:

1. Install the Claude GitHub App: [https://github.com/apps/claude](https://github.com/apps/claude)
2. Add `ANTHROPIC_API_KEY` to your repository secrets
3. Copy the [example workflow](https://github.com/anthropics/claude-code-action/blob/main/examples/claude.yml) into `.github/workflows/`

> **Note**: `/install-github-app` is only available for direct Anthropic API users. AWS Bedrock and Google Vertex AI users must set up manually.

---

## Basic Workflow YAML

```yaml
name: Claude Code
on:
  issue_comment:
    types: [created]
  pull_request_review_comment:
    types: [created]
jobs:
  claude:
    runs-on: ubuntu-latest
    steps:
      - uses: anthropics/claude-code-action@v1
        with:
          anthropic_api_key: ${{ secrets.ANTHROPIC_API_KEY }}
          # Responds to @claude mentions in comments
```

Once set up, trigger Claude from any issue or PR comment:
```
@claude implement this feature based on the issue description
@claude review this PR for security issues
@claude fix the TypeError in the user dashboard component
```

---

## Common Use Cases

### Automated PR Code Review

```yaml
name: Code Review
on:
  pull_request:
    types: [opened, synchronize]
jobs:
  review:
    runs-on: ubuntu-latest
    steps:
      - uses: anthropics/claude-code-action@v1
        with:
          anthropic_api_key: ${{ secrets.ANTHROPIC_API_KEY }}
          prompt: "Review this pull request for code quality, correctness, and security. Post your findings as review comments."
          claude_args: "--max-turns 5"
```

### Scheduled Automation

```yaml
name: Daily Report
on:
  schedule:
    - cron: "0 9 * * *"
jobs:
  report:
    runs-on: ubuntu-latest
    steps:
      - uses: anthropics/claude-code-action@v1
        with:
          anthropic_api_key: ${{ secrets.ANTHROPIC_API_KEY }}
          prompt: "Generate a summary of yesterday's commits and open issues"
          claude_args: "--model opus"
```

---

## Action Parameters (v1.0)

| Parameter | Description | Required |
|-----------|-------------|----------|
| `prompt` | Instructions for Claude (plain text or a skill name) | No* |
| `claude_args` | CLI arguments passed to Claude Code | No |
| `anthropic_api_key` | Claude API key | Yes** |
| `github_token` | GitHub token for API access | No |
| `trigger_phrase` | Custom trigger phrase (default: `@claude`) | No |
| `use_bedrock` | Use AWS Bedrock instead of Claude API | No |
| `use_vertex` | Use Google Vertex AI instead of Claude API | No |

\*Prompt is optional — when omitted for issue/PR comments, Claude responds to trigger phrase  
\*\*Required for direct Claude API, not for Bedrock/Vertex

### Common `claude_args`

```yaml
claude_args: "--max-turns 5 --model claude-sonnet-4-6 --allowedTools Edit,Bash"
```

---

## Using with AWS Bedrock

```yaml
name: Claude PR Action
permissions:
  contents: write
  pull-requests: write
  issues: write
  id-token: write
on:
  issue_comment:
    types: [created]
jobs:
  claude-pr:
    if: contains(github.event.comment.body, '@claude')
    runs-on: ubuntu-latest
    env:
      AWS_REGION: us-west-2
    steps:
      - uses: actions/checkout@v4

      - name: Generate GitHub App token
        id: app-token
        uses: actions/create-github-app-token@v2
        with:
          app-id: ${{ secrets.APP_ID }}
          private-key: ${{ secrets.APP_PRIVATE_KEY }}

      - name: Configure AWS Credentials (OIDC)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_TO_ASSUME }}
          aws-region: us-west-2

      - uses: anthropics/claude-code-action@v1
        with:
          github_token: ${{ steps.app-token.outputs.token }}
          use_bedrock: "true"
          claude_args: '--model us.anthropic.claude-sonnet-4-6 --max-turns 10'
```

> **Bedrock model IDs** include a region prefix, e.g. `us.anthropic.claude-sonnet-4-6`.

---

## Using with Google Vertex AI

```yaml
name: Claude PR Action
permissions:
  contents: write
  pull-requests: write
  issues: write
  id-token: write
on:
  issue_comment:
    types: [created]
jobs:
  claude-pr:
    if: contains(github.event.comment.body, '@claude')
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Generate GitHub App token
        id: app-token
        uses: actions/create-github-app-token@v2
        with:
          app-id: ${{ secrets.APP_ID }}
          private-key: ${{ secrets.APP_PRIVATE_KEY }}

      - name: Authenticate to Google Cloud
        id: auth
        uses: google-github-actions/auth@v2
        with:
          workload_identity_provider: ${{ secrets.GCP_WORKLOAD_IDENTITY_PROVIDER }}
          service_account: ${{ secrets.GCP_SERVICE_ACCOUNT }}

      - uses: anthropics/claude-code-action@v1
        with:
          github_token: ${{ steps.app-token.outputs.token }}
          use_vertex: "true"
          claude_args: '--model claude-sonnet-4-5@20250929 --max-turns 10'
        env:
          ANTHROPIC_VERTEX_PROJECT_ID: ${{ steps.auth.outputs.project_id }}
          CLOUD_ML_REGION: us-east5
```

---

## Upgrading from Beta to v1.0

Key changes when upgrading:

| Old Beta Input | New v1.0 Input |
|----------------|----------------|
| `mode` | *(Removed — auto-detected)* |
| `direct_prompt` | `prompt` |
| `override_prompt` | `prompt` with GitHub variables |
| `custom_instructions` | `claude_args: --append-system-prompt` |
| `max_turns` | `claude_args: --max-turns` |
| `model` | `claude_args: --model` |
| `allowed_tools` | `claude_args: --allowedTools` |
| `disallowed_tools` | `claude_args: --disallowedTools` |

---

## Security Best Practices

- Always use `${{ secrets.ANTHROPIC_API_KEY }}` — never hardcode API keys
- Limit action permissions to only what is necessary
- Use OIDC (not static credentials) for AWS/GCP authentication
- Review Claude's suggestions before merging
- See [security documentation](https://github.com/anthropics/claude-code-action/blob/main/docs/security.md)

---

## Sources

- [GitHub Actions — Claude Code Docs](https://code.claude.com/docs/en/github-actions)
- [claude-code-action repository](https://github.com/anthropics/claude-code-action)
- [GitHub Actions examples directory](https://github.com/anthropics/claude-code-action/tree/main/examples)
