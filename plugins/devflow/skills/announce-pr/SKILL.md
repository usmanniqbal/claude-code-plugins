---
name: announce-pr
description: Announce a PR is ready for review — transition the Jira ticket to "To Review" first, then post to Slack with link previews disabled. Use when the user says to post/announce a PR.
---

# announce-pr

Announce that a PR is ready for review in a way that keeps Jira and Slack consistent.

## Order matters

Always execute in this exact order. Do not swap steps.

### 1. Transition the Jira ticket

Move the ticket to the **"To Review"** board column **before** posting to Slack.

- Target status name: `Review`
- Transition id in the EXP project: **`41`**
- Do **not** use `81` — that maps to "In Review", a different status that lives later in the workflow.

Use the `transitionJiraIssue` tool on the bundled `atlassian` MCP server with the ticket key from the PR title/body. Do this without asking the user — it's part of the workflow.

### 2. Preview the Slack message

Show the user:
- The exact message text
- The target channel (name + ID)

Wait for explicit confirmation. Posting to a team channel is public and hard to reverse.

Message format:
```
EXP-XXXXX: <short description>
<PR URL>
```

### 3. Post to Slack

Use `chat.postMessage` with `unfurl_links: false` to keep the channel clean:

```json
{
  "channel": "CHANNEL_ID",
  "text": "EXP-XXXXX: description\n<GITHUB_URL>",
  "unfurl_links": false
}
```

## Why the order matters

Teammates scanning Slack click through expecting the ticket to already be in "To Review". A posted PR with a ticket still in "In Progress" creates confusion and slows review.

## Do not

- Do not post to Slack before transitioning the ticket.
- Do not omit `unfurl_links: false` — channels get noisy fast with expanded previews.
- Do not post without previewing the message to the user first.
- Do not use transition id `81` (In Review) — that's the wrong status for this step.
