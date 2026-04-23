---
name: review-pr
description: Review a pull request with full Jira context — always fetch the linked ticket before finalizing recommendations. Use when the user asks to review a PR.
---

# review-pr

Review a PR with the customer context, acceptance criteria, and urgency signals that aren't visible in the diff alone.

## Steps

1. **Fetch the PR** with `gh pr view <number> --json ...` including title, body, files, commits, and author.

2. **Extract the Jira ticket reference** from the PR title or body (pattern: `EXP-\d+` or similar project prefixes). If none is present, flag it to the user and ask whether to proceed without ticket context.

3. **Suggest a session rename** as the very first line of your response, on its own line, exactly in this format so the user can one-tap it:

   ```
   /rename Reviewing - <TICKET> - <PR title>
   ```

   Replace `<TICKET>` with the ticket key (e.g., `EXP-12345`) and `<PR title>` with the PR's title. Output this line before any other narration — it's how the user tags the session. Skills cannot invoke slash commands directly; the user must run it themselves.

4. **Fetch the Jira ticket** via the bundled Atlassian MCP (`getJiraIssue` tool on the `atlassian` server). This MCP is declared in the plugin's `.mcp.json` and is guaranteed to be available — if it isn't, tell the user to authenticate it via `/mcp` rather than proceeding without ticket context. Capture:
   - Status, priority, assignee
   - Description and acceptance criteria
   - Linked external tickets (HubSpot escalations, support cases)
   - Affected events or customers mentioned in the ticket
   - Parent epic context if referenced

5. **Follow Slack links from the ticket.** Scan the ticket description and comments for Slack permalinks matching:

   ```
   https://[a-z0-9-]+\.slack\.com/archives/[A-Z0-9]+/p\d+
   ```

   If any are found, this adds real customer-complaint / decision-discussion context the Jira ticket alone rarely captures.

   **Consent flow** (the user controls this, once per session or permanently):

   a. **Check for a standing preference** first — read `~/.claude/projects/-Users-usman-developer/memory/feedback_slack_autofollow.md`. If it exists and indicates "always follow Slack links", skip the prompt and proceed to fetch.

   b. **If no standing preference**, show the user the count and ask:

      > "Found N Slack link(s) in EXP-XXXXX. Fetch them for extra context? Reply `this time`, `always` (save preference), or `no`."

      Wait for explicit answer. On `no`, skip this step. On `always`, save the preference to memory using the structure in `~/.claude/projects/-Users-usman-developer/memory/MEMORY.md` conventions (type: feedback, description of the preference), then proceed.

   c. **Fetch the threads** via the bundled Slack MCP (`slack` server). Parse each permalink into channel ID (`C...`) and timestamp (insert a `.` before the last 6 digits of the `p` number — `p1234567890123456` → `1234567890.123456`), then call the MCP's conversation-replies tool. If the server exposes a direct `get_permalink_message` or equivalent, prefer that.

   d. **Cap at the 3 most-recent links** if the ticket has more, to avoid bloating context. Mention the ones skipped so the user can ask for specific ones if needed.

   e. Summarize each thread: who said what, any customer names, any decisions. Fold into the ticket context for the calibration step.

6. **Read the diff** and identify the changes. For non-trivial changes, read the surrounding code to understand blast radius, not just the hunks.

7. **Calibrate the review against ticket context**:
   - A generic-looking timezone fix may be a customer-blocking issue if the ticket mentions a live event or a HubSpot escalation — adjust urgency accordingly
   - A "P2 cleanup" diff warrants a lighter touch than a P0 regression fix
   - Verify the diff actually satisfies the acceptance criteria, not just that the code is clean

8. **Produce the review** with sections for: summary, correctness, edge cases, tests, and any blocking concerns. Lead with ticket-derived context so the reviewer/author can see the priority framing.

9. **Offer to post the review to GitHub.** After showing the review in chat, ask the user whether to post it. If they decline, stop. If they accept, go to step 10.

10. **Preview the exact payload** before posting. Show the user, in a single message:
   - **Verdict**: `approve` | `request-changes` | `comment` (pick based on the review — default to `comment` unless blockers are present, in which case `request-changes`; only `approve` if explicitly asked)
   - **Overall body**: the summary comment that will appear at the top of the review
   - **Inline comments**: one bullet per comment with `path:line` and the exact body text
   - Target PR URL and branch

   Wait for explicit user confirmation (e.g., "yes, post it"). Do not post on implicit agreement.

11. **Post the review** as a single GitHub review, not as scattered comments. Use the batched review API so everything lands atomically:

    ```bash
    # Start a pending review, add inline comments, submit — one script
    gh api repos/<owner>/<repo>/pulls/<number>/reviews \
      --method POST \
      --input - <<'JSON'
    {
      "event": "COMMENT",              // or "REQUEST_CHANGES" / "APPROVE"
      "body": "<overall summary>",
      "comments": [
        {"path": "src/foo.ts", "line": 42, "body": "..."},
        {"path": "src/bar.ts", "line": 17, "body": "..."}
      ]
    }
    JSON
    ```

    Use `gh pr view --json url,headRefOid,baseRefName` to get the commit SHA if the API rejects without one. Report the review URL on success.

## Do not

- Do not finalize a review without reading the linked Jira ticket.
- Do not assume ticket status from the PR alone — fetch it.
- Do not downgrade urgency just because the diff looks small; the ticket decides the stakes.
- Do not post comments to GitHub without a preview and explicit confirmation — posting is public and hard to reverse.
- Do not post inline comments one at a time via `gh pr comment` — batch them into a single review so the author gets one notification, not N.
- Do not choose `approve` unless the user explicitly asked for an approval; default to `comment` or `request-changes`.
