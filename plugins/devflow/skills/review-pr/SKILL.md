---
name: review-pr
description: Use when the user asks to review a pull request, requests a second-opinion code review, or shares a PR number/URL and wants feedback on the diff
---

# review-pr

Review a PR with the customer context, acceptance criteria, and urgency signals that aren't visible in the diff alone.

**REQUIRED:** Read `../../CONVENTIONS.md` first. It defines tool access, anchored references, and writing style. Apply throughout.

## Steps

1. **Resolve the repo and PR number** per `CONVENTIONS.md` (parse a PR URL if given, else `gh repo view --json nameWithOwner`, else ask). Cache `<owner>/<repo>` and `<n>`. Then fetch: `gh pr view <n> --repo <owner>/<repo> --json title,body,number,headRefName,headRefOid,baseRefName,url,author,files,commits`.

2. **Extract the Jira ticket** from the title or body (`EXP-\d+` or similar). If absent, ask the user before proceeding without ticket context.

3. **Output a session-rename suggestion** as the first line of your response, on its own line, exactly:

   ```
   /rename Reviewing - <TICKET> - <PR title>
   ```

   `<TICKET>` is the ticket key, never the PR number. `<PR title>` is the exact title from `gh pr view --json title`, never the repo or branch name.

   Then follow `CONVENTIONS.md` "Tmux tab renaming" with `<verb>` = `review` (label: `<TICKET> review`).

4. **Fetch the Jira ticket** via the Atlassian MCP (`getJiraIssue`). Capture status, priority, assignee, description, acceptance criteria, linked external tickets (HubSpot escalations, support cases), and parent epic if referenced.

5. **Follow Slack links from the ticket.** Scan ticket description and comments for `https://<workspace>.slack.com/archives/<C...>/p<ts>` permalinks.

   First check `~/.claude/projects/-Users-usman-developer/memory/feedback_slack_autofollow.md` for a standing preference. If absent, ask: "Found N Slack link(s). Fetch them? `this time` / `always` / `no`." On `always`, save the preference to memory.

   Fetch the threads via the Slack MCP (parse `channel` from the `C...` segment, `ts` by inserting a `.` before the last 6 digits of the `p` number). Cap at the 3 most-recent links. Summarize who said what, customer names, decisions; fold into ticket context.

6. **Read the diff** for non-trivial changes; understand blast radius, not just hunks.

7. **Calibrate**: a generic-looking diff is high-stakes if the ticket mentions a live event or escalation. Verify the diff satisfies acceptance criteria, not just code cleanliness. Default `request-changes` for blockers, `comment` otherwise; only `approve` if explicitly asked.

8. **Produce the review** in chat with sections: summary, correctness, edge cases, tests, blockers. Lead with ticket context.

9. **Offer to post to GitHub.** Ask whether to post. If declined, stop.

10. **Preview the exact payload**: verdict, overall body, each inline comment as `path:line` + body, target PR URL. Wait for explicit "yes, post it".

11. **Post as a single batched review** so the author gets one notification:

    ```bash
    gh api repos/<owner>/<repo>/pulls/<n>/reviews --method POST --input - <<'JSON'
    {
      "event": "COMMENT",
      "body": "<overall summary>",
      "comments": [
        {"path": "src/foo.ts", "line": 42, "body": "..."}
      ]
    }
    JSON
    ```

    Report the review URL on success.

## Do not

- Finalize without reading the linked Jira ticket.
- Post comments to GitHub without preview and explicit confirmation.
- Post inline comments one at a time via `gh pr comment`; batch into one review.
- Choose `approve` unless the user explicitly asked.
