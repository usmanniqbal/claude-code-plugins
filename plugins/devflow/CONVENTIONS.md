# devflow conventions

Shared rules referenced by every skill in this plugin. Read this once per session, then apply throughout.

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

Whenever you mention a file, line, review comment, Jira ticket, fixup commit, DeepSource issue, or Slack thread in chat output, format it as a markdown link. Never bare IDs or bare URLs. Apply everywhere outside code fences.

- **File + line within the PR** (preferred, opens Files Changed with the diff visible):
  - `[path/file.ts:42](https://github.com/<owner>/<repo>/pull/<n>/files#diff-<hash>R42)`
  - `<hash>` = SHA-256 of the path: `echo -n "path/file.ts" | shasum -a 256 | cut -d' ' -f1`
  - `R<line>` = right (new) side, `L<line>` = left (old). Reviews almost always use `R`.
  - PR-files anchors don't support multi-line ranges; for ranges, link the first line.
- **File + line outside the PR** (fallback):
  - `[path/file.ts:42](https://github.com/<owner>/<repo>/blob/<sha>/path/file.ts#L42)`
  - `<sha>` = `headRefOid` from `gh pr view --json headRefOid`. Multi-line: `#L42-L58`.
- **Jira ticket**: `[EXP-22217](https://<org>.atlassian.net/browse/EXP-22217)`. Get `<org>` once per session via the Atlassian MCP's `getAccessibleAtlassianResources`, or pull from any existing Jira link in the PR. Cache.
- **Existing PR review comment**: `https://github.com/<owner>/<repo>/pull/<n>#discussion_r<comment-id>`, or use the `html_url` returned by the GitHub API for newly-posted comments.
- **DeepSource issue**: prefer the URL the CLI returns; fallback `https://app.deepsource.com/gh/<owner>/<repo>/issue/<id>/`.
- **Fixup commit**: `[fixup abc1234](https://github.com/<owner>/<repo>/commit/abc1234)`.
- **Slack thread**: link the permalink directly: `[customer thread](https://<workspace>.slack.com/archives/C.../p...)`.

## Writing style

- No em dashes anywhere in chat output, commit messages, PR bodies, or replies. Use commas, colons, parentheses, or periods.
- Conventional Commits with a `Resolves: <TICKET>` footer when there's a Jira ticket.
- PR iteration uses `git commit --fixup <sha>`, never `--amend`. One fixup per main commit.

## Expected-fail CI checks

These two checks legitimately fail while the branch has unsquashed `fixup!` commits and pass at merge. Filter them out of any "failing checks" list:

- `Commit / Autosquash check (...)`
- `Commit / Commit message lint (...)`
