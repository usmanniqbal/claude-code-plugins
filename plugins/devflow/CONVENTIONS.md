# devflow conventions

Shared rules referenced by every skill in this plugin. Read this once per session, then apply throughout.

## Repo resolution

Every skill that touches a PR needs to know which repo (`<owner>/<repo>`) and PR number it's working with. Resolve in this order:

1. **Explicit input.** If the user passed a PR URL (`https://github.com/<owner>/<repo>/pull/<n>`), parse it; that's authoritative. If they passed just a number (`#861` or `861`), fall through to the cwd.
2. **Current working directory.** Run `gh repo view --json nameWithOwner,defaultBranchRef -q '.nameWithOwner'`. If it succeeds, you're in a clone of that repo. Combine with the PR number from step 1.
3. **No repo context.** If neither yields a result (cwd is not a git repo, or `gh` isn't auth'd), ask the user: "Which repo is PR #<n> in? (`<owner>/<repo>`)." Don't guess.

Once resolved, cache `<owner>`, `<repo>`, and `<n>` for the rest of the session. Every `gh` and `gh api` call in the skill should pass `--repo <owner>/<repo>` explicitly so the skill works from any cwd, not just inside the repo's clone.

**Default branch:** never assume `main` or `master`. Resolve once per session: `gh repo view <owner>/<repo> --json defaultBranchRef -q '.defaultBranchRef.name'`. Cache as `<base>` and use it for diff ranges, base-branch comparisons, and any "reset to default" instructions.

If the user refers to a product or app name instead of a repo (for example, "the mobile app PR" or "review the portal"), check user memory for a repo-to-product mapping before asking. If no mapping is recorded, ask which repo they mean.

## Tool access preference

When a skill needs data from an external service (Jira, Slack, GitHub, Figma, Notion, DeepSource, etc.), prefer methods in this order:

1. **MCP** (structured, typed, OAuth-managed).
2. **Official CLI** (`gh`, `jira`, `deepsource`, `slack-cli`, `figma`, etc.).
3. **curl** against the public API (last resort).

**Never fall back silently between tiers.** At each transition, ask:

- **No MCP connected?** Ask: "No <service> MCP is connected. Install and connect one now, or fall back to the CLI?"
  On install: name the recommended server (for example, official Atlassian remote MCP at `https://mcp.atlassian.com/v1/mcp`, or the `github` plugin from `claude-plugins-official`). Provide the exact `/plugin install` or `.mcp.json` edit, prompt for `/mcp` auth, then `/reload-plugins`. Resume once connected.
- **No CLI on PATH?** Ask: "`<cli>` is not installed for <service>. Install it now, or fall back to curl?"
  On install: give the exact install command, verify the binary is on PATH, resume.
- **Only curl left?** Use it, acknowledging it is the last-resort path.

## Anchored references

Whenever you mention any external resource (files, lines, tickets, comments, threads, pages, frames, dashboards, etc.) in chat output, format it as a markdown link to the most specific place a click can land. Never bare IDs or bare URLs. Apply everywhere outside code fences.

**The rule of specificity:** link to the smallest addressable unit you have. If you're talking about a comment on a Jira ticket, link the comment, not the ticket. If you're talking about a Figma node, link the node, not the file. If you're talking about a paragraph in Notion, link the block, not the page.

### GitHub

- **File + line within the PR** (preferred, opens Files Changed with the diff visible):
  - `[path/file.ts:42](https://github.com/<owner>/<repo>/pull/<n>/files#diff-<hash>R42)`
  - `<hash>` = SHA-256 of the path: `echo -n "path/file.ts" | shasum -a 256 | cut -d' ' -f1`
  - `R<line>` = right (new) side, `L<line>` = left (old). Reviews almost always use `R`.
  - PR-files anchors don't support multi-line ranges; for ranges, link the first line.
- **File + line outside the PR** (fallback):
  - `[path/file.ts:42](https://github.com/<owner>/<repo>/blob/<sha>/path/file.ts#L42)`
  - `<sha>` = `headRefOid` from `gh pr view --json headRefOid`. Multi-line: `#L42-L58`.
- **PR review comment**: `https://github.com/<owner>/<repo>/pull/<n>#discussion_r<comment-id>`, or the `html_url` returned by the API for newly-posted comments.
- **PR top-level comment**: `https://github.com/<owner>/<repo>/pull/<n>#issuecomment-<comment-id>`.
- **PR commit**: `https://github.com/<owner>/<repo>/pull/<n>/commits/<sha>` (or `/commit/<sha>` for repo-level).
- **Issue**: `[#1234](https://github.com/<owner>/<repo>/issues/1234)`. For a specific comment, append `#issuecomment-<id>`.
- **Workflow run / failing check**: link the specific run, e.g., `[lint failed](https://github.com/<owner>/<repo>/actions/runs/<run-id>)`. For a job, append `/job/<job-id>`.

### Jira (Atlassian)

- **Ticket**: `[EXP-22217](https://<org>.atlassian.net/browse/EXP-22217)`. Get `<org>` once per session via the Atlassian MCP's `getAccessibleAtlassianResources`, or pull from any existing Jira link in the PR. Cache.
- **Ticket comment**: `https://<org>.atlassian.net/browse/EXP-22217?focusedCommentId=<comment-id>`. The Atlassian MCP returns comment IDs in `getJiraIssue`'s comment payload.
- **Ticket activity tab** (history, worklog): append `?focusedTab=<tab>` such as `comments`, `history`, `worklog`.
- **Confluence page**: use the page URL as returned by the MCP; for an anchor inside a page, append `#<heading-anchor>`.

### Slack

- **Message or thread**: link the permalink directly. `[customer thread](https://<workspace>.slack.com/archives/<C...>/p<ts>)`. The `p` form anchors to the exact message; without `p` it just opens the channel.
- **Channel**: only when explicitly referring to the channel itself, not a message inside it.

### Notion

- **Page**: `[Page title](https://www.notion.so/<workspace>/<page-id>)`. The Notion MCP returns full page URLs.
- **Block within a page**: append `#<block-id>` to land on a specific paragraph, heading, or callout. The MCP exposes block IDs.
- **Database row**: `https://www.notion.so/<workspace>/<row-page-id>`.

### Figma

- **File**: `[Designs](https://www.figma.com/file/<file-key>/<file-name>)`.
- **Frame or node** (preferred when discussing specific UI): append `?node-id=<node-id>`. Format: `https://www.figma.com/file/<file-key>/<file-name>?node-id=123-456`. The Figma MCP exposes node IDs.
- **Comment on a frame**: append `#<comment-id>` after the node-id query.

### Other

- **DeepSource issue**: prefer the URL the CLI returns; fallback `https://app.deepsource.com/gh/<owner>/<repo>/issue/<id>/`.
- **Fixup commit**: `[fixup abc1234](https://github.com/<owner>/<repo>/commit/abc1234)`.
- **Sentry / Datadog / Grafana / monitoring dashboard**: always include the deep-link URL with the relevant query params (time range, filters) preserved, so a click reproduces the view you saw.
- **Anything else with a URL**: link it. If the source has a "copy link to this comment / block / frame / row" affordance, that's the URL to use.

When the most-specific URL isn't easily available, link the closest level you do have, and say so briefly (for example, "Couldn't get a comment-deep link, here's the ticket: [EXP-22217](...)"). Never silently drop to a less-specific link.

## Writing style

- No em dashes anywhere in chat output, commit messages, PR bodies, or replies. Use commas, colons, parentheses, or periods.
- Conventional Commits with a `Resolves: <TICKET>` footer when there's a Jira ticket.
- PR iteration uses `git commit --fixup <sha>`, never `--amend`. One fixup per main commit.

## Expected-fail CI checks

These two checks legitimately fail while the branch has unsquashed `fixup!` commits and pass at merge. Filter them out of any "failing checks" list:

- `Commit / Autosquash check (...)`
- `Commit / Commit message lint (...)`
