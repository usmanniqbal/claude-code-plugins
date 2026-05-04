---
name: announce-pr
description: Use when the user says "announce the PR", "post the PR to Slack", "share the PR with the team", or asks to mark a PR ready for review
---

# announce-pr

Announce a PR is ready for review. Keep Jira and Slack consistent: transition the ticket first, then post.

**REQUIRED:** Read `../../CONVENTIONS.md` first. It defines tool access, anchored references, and writing style.

## Steps, in order

### 0. Resolve the PR

Per `CONVENTIONS.md`: parse a PR URL if given, else `gh pr view --json url,number,title,body --repo $(gh repo view --json nameWithOwner -q .nameWithOwner)` from the cwd, else ask. Cache the PR URL, number, and ticket key from the title.

### 1. Transition the Jira ticket first

Move the ticket to **"To Review"** before posting anywhere.

- Status name: `Review`.
- Transition id in the EXP project: **`41`**. Do not use `81` ("In Review", a different status later in the workflow).

Use the Atlassian MCP's `transitionJiraIssue` with the ticket key from the PR title or body. Do this without asking; it's part of the workflow.

### 2. Preview the Slack message

Show the user the exact message text and target channel (name and ID). Wait for explicit confirmation. Posting to a team channel is public and hard to reverse.

Format:

```
EXP-XXXXX: <short description>
<PR URL>
```

### 3. Post to Slack

Use `chat.postMessage` (via the Slack MCP) with `unfurl_links: false` to keep the channel clean.

```json
{
  "channel": "CHANNEL_ID",
  "text": "EXP-XXXXX: description\n<GITHUB_URL>",
  "unfurl_links": false
}
```

## Why the order

Teammates clicking through Slack expect the ticket to already be in "To Review". A posted PR with a ticket still "In Progress" creates confusion and slows review.

## Do not

- Post to Slack before transitioning the ticket.
- Omit `unfurl_links: false`; channels get noisy with previews.
- Post without previewing the message to the user first.
- Use transition id `81` (In Review); that's the wrong status.
