---
name: start-ticket
description: Use when the user shares a Jira ticket URL or key and wants to start work, plan an approach, or research what's involved before coding
---

# start-ticket

Turn a Jira ticket into a grounded research summary and a draft plan: scope, related context, branch and worktree suggestion, files likely touched, open questions. Stop before any code is written.

**REQUIRED:** Read `../../CONVENTIONS.md` first. It defines tool access (MCP > CLI > curl), anchored references, repo and default-branch resolution, and writing style.

## Steps

1. **Resolve the ticket.** Parse the Jira key from the URL or argument (e.g., `EXP-22217`). If the user gave a vague reference (a sentence, a screenshot, a Slack message), ask for the explicit key or URL. Don't guess.

2. **Fetch the ticket via Atlassian MCP** (`getJiraIssue`). If no Atlassian MCP is connected, follow the tool-access escalation in `CONVENTIONS.md`. Pull at minimum: summary, description, status, priority, assignee, reporter, labels, components, custom fields (sprint, team, story points, epic link), and the full comment thread.

3. **Resolve the target repo.** Look in this order, stop at the first hit:
   - Components or labels on the ticket that map to a repo (check user memory for any product-to-repo mapping).
   - URLs in the description or comments that point to a GitHub repo.
   - Linked PRs returned by the Jira issue's development panel.
   - Ask the user.

   Cache `<owner>/<repo>` and `<base>` (default branch via `gh repo view --json defaultBranchRef`) per `CONVENTIONS.md`.

4. **Suggest a session rename** as the next line of your response, on its own line:

   ```
   /rename Planning - <TICKET> - <short desc>
   ```

   `<short desc>` = 3-6 words derived from the ticket title (skip filler like "fix", "update", "the").

5. **Walk linked resources, one hop only.** For every external link in the ticket's description and comments, fetch the target via the appropriate MCP per `CONVENTIONS.md`:
   - **Figma**: `get_design_context` and `get_screenshot` for each linked node.
   - **Slack**: fetch the linked message or thread permalink.
   - **Notion**: fetch the page (and the specific block if anchored).
   - **GitHub**: linked PRs, issues, files, runs, commits.
   - **Other tickets** (linked Jira keys): summary + status only, do not recurse.
   - **Dashboards** (Grafana, Datadog, Sentry): note the URL and timeframe; don't follow further.

   **Hard stop at one hop.** Do not follow links found *inside* the linked resources. If a Slack thread links three more docs, list them as "follow-up references" and move on. The point is grounding, not exhaustive crawling.

6. **Survey likely-affected files** (rough, not exhaustive). Use `git ls-files` and `grep` against `<base>` of the resolved repo for terms from the ticket: component names, function names, feature flags, route paths. 5-10 candidate files is plenty.

7. **Produce the research summary.** Anchored references throughout (per `CONVENTIONS.md`). Sections:
   - **Ticket**: link, status, assignee, priority, sprint.
   - **Ask**: 1-3 sentences on what's being requested.
   - **Acceptance criteria**: pulled verbatim from the description; flag explicitly if missing.
   - **Customer / stakeholder context**: who asked, why, deadline if any.
   - **Design**: Figma frames seen, with screenshots if UI-relevant.
   - **Prior art**: related PRs, similar past tickets, ADRs.
   - **Linked discussions**: Slack threads, Notion docs, ticket comments worth reading.
   - **Open questions**: gaps in the ticket the user should resolve before coding.

8. **Draft the plan.** Sections:
   - **Branch name**: `<TICKET>-<kebab-desc>`, 3-5 words derived from the ticket title (no generic placeholders).
   - **Worktree location** (if the user uses worktrees): a directory whose final segment matches the branch name, so `cd <path>` lands without a lookup. Follow the user's worktree convention if one is recorded in user memory or in this repo's CLAUDE.md; ask if neither is set. Don't hardcode a parent directory.
   - **Approach**: 3-7 bullets on the plan of attack, grounded in the surveyed files and design.
   - **Files likely touched**: with anchored links to the current `<base>` versions.
   - **Risks / unknowns**: what could derail this, what needs investigation first.

9. **Stop.** Present the summary and plan, then ask:

   > "Approve the plan, refine it, or start coding? On approval I'll create the worktree and branch from latest `<base>`."

   Wait for explicit approval. No worktree, no branch, no edits until the user says go.

## Do not

- Start coding, create branches, or open worktrees before explicit approval.
- Recurse past one link hop. Note follow-up references and move on.
- Transition the Jira ticket, comment on it, or update fields. This skill is read-only on Jira.
- Skim the description and skip the comments. Comments often hold the latest scope, blockers, and stakeholder asks.
- Substitute generic placeholders ("fix", "wip", "update") in the branch name. The description must reflect the ticket.
- Invent acceptance criteria. If the ticket lacks them, surface that as an open question.
- Anchor a UI plan to memory or convention alone when the ticket has Figma. Pull the design context first.
