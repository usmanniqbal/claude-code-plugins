# claude-code-plugins

Claude Code plugins for PR workflow: create, review, announce, and address comments. Bundles Atlassian and Slack remote MCPs.

## Install

In Claude Code:

```
/plugin marketplace add usmanniqbal/claude-code-plugins
/plugin install devflow@usmanniqbal
```

Plugin skills are namespaced, invoked as `/devflow:start-ticket`, `/devflow:create-pr`, `/devflow:review-pr`, `/devflow:announce-pr`, `/devflow:address-pr-comments`, `/devflow:dependabot-triage`, and `/devflow:announce-dependabot-merges`.

### First-time MCP auth

The `devflow` plugin bundles two official remote MCP servers so the skills work out of the box:

- **Atlassian** (`https://mcp.atlassian.com/v1/mcp`): used by `start-ticket` (fetch the ticket and its comments), `review-pr` (fetch Jira ticket), and `announce-pr` (transition to Review).
- **Slack** (`https://mcp.slack.com/mcp`): used by `start-ticket` and `review-pr` to follow Slack permalinks from tickets for customer/decision context, and by `announce-pr` to post the PR message.

On first use, run `/mcp` and authenticate both servers via OAuth.

## Plugins

### devflow

| Skill | What it does |
|---|---|
| `start-ticket` | Fetches a Jira ticket and walks linked Figma/Slack/Notion/GitHub resources one hop, then produces a research summary, suggested branch and worktree, files likely touched, and open questions. Stops before any code is written. |
| `create-pr` | Opens a PR using the repo's `.github/pull_request_template.md`, verifies referenced configs and packages exist, writes realistic verification steps. |
| `review-pr` | Reviews a PR with the linked Jira ticket's context: customer impact, acceptance criteria, linked escalations, optional Slack thread follow-up. |
| `announce-pr` | Transitions the Jira ticket to "To Review" (status `Review`, transition id `41`), then posts to Slack with `unfurl_links: false` after previewing. |
| `address-pr-comments` | Triages review threads, failing CI checks, and DeepSource issues. Categorizes each as fix/reply/defer, previews the plan, and on approval pushes fixup commits and posts replies. Skips the autosquash and commit-message-lint checks that expectedly fail on fixups. |
| `dependabot-triage` | Researches open Dependabot PRs: CI rollup, mergeability, branch protection (classic or rulesets), per-package version bumps and changelogs, staleness, and false-⛔ CI-infra failures. Categorizes ✅/🟡/⛔ with a merge order. Read-only; never merges or comments without per-PR approval. |
| `announce-dependabot-merges` | Drafts a Slack digest of the Dependabot PRs merged across the EventMobi frontend repos over a time window, in the team's house mrkdwn style. Drafts only; never posts to a channel. |

All skills read `plugins/devflow/CONVENTIONS.md` for shared rules: tool access preference (MCP > CLI > curl, ask before each fallback), anchored references, writing style, and expected-fail CI checks.

## Repo layout

```
.claude-plugin/
  marketplace.json              # lists available plugins
plugins/
  devflow/
    .claude-plugin/plugin.json
    .mcp.json                   # bundles Atlassian + Slack remote MCPs
    CONVENTIONS.md              # shared rules referenced by every skill
    skills/
      start-ticket/SKILL.md
      create-pr/SKILL.md
      review-pr/SKILL.md
      announce-pr/SKILL.md
      address-pr-comments/SKILL.md
      dependabot-triage/SKILL.md
      announce-dependabot-merges/SKILL.md
```

## Adding a new plugin

1. Create `plugins/<name>/.claude-plugin/plugin.json`.
2. Add one or more `skills/<skill-name>/SKILL.md` files with frontmatter (`name`, `description`).
3. Register the plugin in `.claude-plugin/marketplace.json`.
