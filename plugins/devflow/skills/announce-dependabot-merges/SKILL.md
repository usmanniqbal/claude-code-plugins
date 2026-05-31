---
name: announce-dependabot-merges
description: Use when the user wants to announce or summarize the Dependabot PRs merged across the EventMobi frontend repos over a time window, typically after an on-call Dependabot triage round, as a Slack digest formatted in the team's house mrkdwn style.
---

# announce-dependabot-merges

Draft a Slack digest of the Dependabot PRs merged across the EventMobi **frontend** repos over a time window, formatted in the team's house mrkdwn style. **Output the ready-to-paste message and STOP** — never post to a channel. Offer to save it as a personal draft only if the user explicitly asks.

**REQUIRED:** Read `../../CONVENTIONS.md` first. It defines tool access (MCP > CLI > curl) and anchored references. Apply throughout.

Inputs (both optional): a **time window** (default: last 7 days) and a **target channel** (default `#frontier`). The user may pass either, e.g. "since 2026-05-19" or "in #frontend-eng".

## 1. Gather the merged PRs (read-only)

FE repos under the `EventMobi` org: `mobileapp`, `experience`, `attendee-portal`, `onsite`, `registration`, `toolkit`, `frontend-standards`.

Search across the org in one shot:

```bash
gh search prs "author:app/dependabot" --owner EventMobi --merged \
  --merged-at ">=<START_DATE>" --limit 80 \
  --json repository,number,title,url,closedAt
```

- `mergedAt` is NOT a valid `gh search prs` field — use `closedAt` (merged PRs are closed).
- Resolve `<START_DATE>` from the window arg. There's no reliable wall clock, so if the window is relative ("7 days"), get today from `date +%F` in a Bash call rather than guessing, then subtract.
- Filter results to the FE repos above; drop any non-FE org repos.

## 2. Format the message (Slack-native mrkdwn)

Slack does NOT render markdown tables, headers, or `[text](url)` links. Use the house style:

- `*single-asterisk bold*` for the title and repo names (NOT `**double**`).
- Anchored links in Slack syntax: `<https://github.com/EventMobi/<repo>/pull/<n>|#<n>>` (e.g. `<https://github.com/EventMobi/experience/pull/6577|build: group Vite entrypoints>`).
- `   • ` bullets, grouped per repo.

Clean each PR title into a short label:

- strip the `chore(deps): bump ` / `chore(deps-dev): bump ` prefix
- `X from A to B ...` → `X A→B`
- `the X group across 1 directory with N updates` → `X group (N pkgs)`

Structure (mention ONLY what was merged — do NOT list or explain what wasn't merged):

```
*🔧 Dependabot cleanup — <N> dependency PRs merged across FE repos this week*

Caught up on the on-call Dependabot backlog. Merged across our frontend repos:

*<repo>*
   • <url|#NNNN> <label>
   ...
```

No closing line about held-back / unmerged PRs or reasoning.

## 3. Stop

Print the message in a code block so the user can copy it verbatim, name the target channel, and stop. Offer to save it via the Slack MCP as a **personal draft** (`slack_send_message_draft`) only if the user explicitly asks — never post to a channel.

## Do not

- Post to any Slack channel. This skill drafts only; the user posts it themselves.
- Use `**double-asterisk**` bold or `[text](url)` links — Slack mrkdwn uses `*single*` and `<url|text>`.
- List or explain PRs that were NOT merged; the digest is a summary of what landed.
- Infer the window from commit timestamps; resolve `<START_DATE>` from `date +%F` minus the window.
- Use `mergedAt` in `gh search prs`; it's not a valid field — use `closedAt`.
