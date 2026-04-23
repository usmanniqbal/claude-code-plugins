---
name: address-pr-comments
description: Triage open PR review comments and failing CI checks, then either fix the code or draft a reply for each — all previewed and approved before anything is pushed or posted. Use when the user asks to "address PR feedback", "respond to review comments", or "fix the PR comments".
---

# address-pr-comments

Go through unresolved review threads, unanswered PR comments, and failing checks on a PR. For each one, decide whether it needs a code fix or a written reply, preview the plan to the user, and only act on explicit approval.

## Steps

1. **Fetch PR state** in parallel where possible:
   - `gh pr view <number> --json title,body,number,headRefName,baseRefName,url,author,statusCheckRollup,comments,reviews`
   - Unresolved review threads via GraphQL (thread IDs are required to reply/resolve):

     ```bash
     gh api graphql -F owner=<owner> -F name=<repo> -F number=<n> -f query='
       query($owner:String!,$name:String!,$number:Int!){
         repository(owner:$owner,name:$name){
           pullRequest(number:$number){
             reviewThreads(first:100){
               nodes{
                 id isResolved isOutdated
                 comments(first:50){ nodes{ id databaseId path line body author{login} } }
               }
             }
           }
         }
       }'
     ```

2. **Extract the Jira ticket reference** from the PR title or body (pattern: `EXP-\d+` or similar project prefixes). Check the branch name as a fallback — branches are typically named after the ticket (e.g., `EXP-22217`). If no ticket can be found anywhere, ask the user for it rather than proceeding without one.

3. **Check PR ownership before proceeding.** Compare the PR author to the current GitHub user:

   ```bash
   gh api user --jq .login     # current user's login
   # PR author login is already in the gh pr view --json author result
   ```

   If the current user is **not** the PR author, stop and ask explicitly:

   > "This PR was authored by @<author>, not you. Normally only the author addresses their own PR comments. Have they handed this off to you? Confirm to proceed."

   Wait for explicit confirmation ("yes", "they handed it off", etc.). Do not continue to triage, fixes, or replies until the user confirms. If the user is the author, skip this step silently.

4. **Suggest a session rename** as the next line of your response, on its own line, exactly in this format so the user can one-tap it:

   ```
   /rename Addressing - <TICKET> - <PR title>
   ```

   - `<TICKET>` is the ticket key from step 2 (e.g., `EXP-22217`), **never** the PR number.
   - `<PR title>` is the **exact PR title** returned by `gh pr view --json title`. Do not substitute the repo name, the branch name, or an abbreviation. Do not shorten it unless it exceeds ~80 chars; if it does, truncate at a word boundary and append `…`.
   - Example of a good suggestion: `/rename Addressing - EXP-22217 - Fix check-in badge spacing on iPad`
   - Example of a bad suggestion: `/rename Addressing - PR #861 - onsite` (uses PR number and repo name instead of ticket and title)

   Skills cannot invoke slash commands directly; the user must run it themselves.

5. **Build the triage list.** One entry per:
   - Unresolved review thread (inline code comment)
   - Top-level PR comment that hasn't been responded to by the author
   - Failing check in `statusCheckRollup` — **but filter out the expected-fail checks listed in the next step first**

6. **Filter expected-fail CI checks.** The following checks legitimately fail while the branch has unsquashed `fixup!` commits and will pass at merge — **do not triage them, do not mention them as blockers**:

   - `Commit / Autosquash check (...)` — checks for pending fixups; fails by design while any exist.
   - `Commit / Commit message lint (...)` — enforces Conventional Commits per commit; fails on `fixup!` prefixes.

   Skip both silently unless the PR is about to be merged via a non-autosquash path (rare).

7. **Handle DeepSource check failures specially.** If any check from DeepSource is failing, fetch the issues via the CLI rather than trying to read the check log:

   a. **Check auth**: `deepsource auth status`. If not authenticated, stop and tell the user to run:

      ```bash
      deepsource auth login
      ```

      Wait for them to confirm before proceeding. (Install if missing: `curl -fsSL https://cli.deepsource.com/install | sh`.)

   b. **Fetch PR issues** as JSON:

      ```bash
      deepsource issues --pr <number> --output json
      ```

      Optional filters when the list is long: `--severity critical|major|minor`, `--analyzer <name>`, `--category bug-risk|security|performance|...`, `--path <glob>`.

   c. **Parse each DeepSource issue** into a triage item. Each issue has a file path, line range, severity, analyzer, category, and description. Add them to the triage list as `FIX (DeepSource)` entries alongside the review threads and other CI failures.

   d. **After pushing fixes**, re-analysis triggers automatically on the new commit — no manual command needed. Confirm the new status with `gh pr checks <number>` and `deepsource runs --limit 1 --output json` if you need the latest run's state.

8. **Categorize each triage item** as exactly one of:
   - `FIX (code)` — a concrete code change (bug, typo, missing test, refactor the reviewer requested). Identify the exact file(s) and line(s).
   - `FIX (CI)` — a failing check with a clear cause (lint/type/test). Read the check's log via `gh run view <run-id> --log-failed` to confirm the cause.
   - `FIX (DeepSource)` — an issue surfaced by `deepsource issues`. Use the analyzer/category to decide the fix.
   - `REPLY` — a question, subjective preference, discussion, or a concern already handled elsewhere. No code change needed.
   - `DEFER` — ambiguous; needs user input before you can act. Call these out explicitly.

9. **Draft the action for each non-deferred item**:
   - `FIX`: locate the code, plan the edit. Do not edit yet.
   - `REPLY`: draft the exact reply text. Acknowledge the reviewer's point, explain why no change is needed (or point at the commit that already addresses it), keep it concise.
   - Every `FIX` that corresponds to a review thread also gets a short reply ("Fixed in <fixup-sha>.") so the thread shows engagement. DeepSource issues don't need a thread reply — the check going green is the signal.

10. **Preview the full plan to the user** in one message:

   ```
   Item 1 — thread on src/foo.ts:42 (reviewer: @alice)
     Category: FIX (code)
     Edit:    src/foo.ts:42 — <one-line diff summary>
     Reply:   "Fixed in fixup — thanks for catching this."

   Item 2 — CI check: lint
     Category: FIX (CI)
     Cause:   unused import in src/bar.ts
     Edit:    src/bar.ts:3 — remove unused `foo` import

   Item 3 — DeepSource: JS-0128 on src/baz.ts:17 (major, bug-risk)
     Category: FIX (DeepSource)
     Edit:    src/baz.ts:17 — <one-line fix summary>

   Item 4 — thread on src/qux.ts:10 (reviewer: @bob)
     Category: REPLY
     Reply:   "Intentional — this mirrors the pattern in qux.ts and keeps the API symmetric."

   Skipped (expected-fail on fixups):
     - Commit / Autosquash check
     - Commit / Commit message lint

   Summary: 3 fixes → 1 fixup commit targeting <main-sha>, 1 reply.
   ```

   Wait for explicit approval. The user may approve all, a subset, or request edits to the plan. Do not act on silence.

11. **Apply code fixes** (only approved items):
   - Make the edits.
   - Show the final diff to the user before staging (per repo convention — always review before commit).
   - Stage specific files by name. Do not use `git add -A` or `git add .`.
   - Create fixup commits with `git commit --fixup <sha>` targeting the original main commit each fix relates to. One fixup per main commit if the changes span multiple.
   - Push with `git push`.

12. **Post replies and resolve threads** (only approved items):
    - For unresolved review threads, reply via GraphQL and resolve when the fix has been pushed:

      ```bash
      # Reply to a thread
      gh api graphql -F threadId=<thread-id> -F body="..." -f query='
        mutation($threadId:ID!,$body:String!){
          addPullRequestReviewThreadReply(input:{pullRequestReviewThreadId:$threadId,body:$body}){
            comment{ id url }
          }
        }'

      # Resolve the thread (only after the fixup is pushed)
      gh api graphql -F threadId=<thread-id> -f query='
        mutation($threadId:ID!){
          resolveReviewThread(input:{threadId:$threadId}){ thread{ id isResolved } }
        }'
      ```

    - For top-level PR comments: `gh pr comment <number> --body "..."`.
    - For CI-only or DeepSource-only fixes with no associated thread: no reply needed — the check going green is the signal.

13. **Report back**: list of fixup commits (with SHAs + URLs), threads replied-to/resolved, and the new CI status (`gh pr checks <number>`).

## Do not

- Do not push code or post replies without preview + explicit approval. These are public, hard-to-reverse actions.
- Do not proceed past step 3 if the current user is not the PR author without an explicit handoff confirmation — barging into someone else's PR is a social misstep.
- Do not amend prior commits or rebase — use fixup commits only (per repo convention).
- Do not triage `Commit / Autosquash check` or `Commit / Commit message lint` as failures — they're expected to fail on fixup commits and pass at merge.
- Do not try to read DeepSource issues from the check run log — use the `deepsource` CLI instead.
- Do not resolve a review thread unless the fix has actually been pushed; an unpushed local fix doesn't count.
- Do not mass-reply "fixed!" to threads you didn't actually fix. Each reply should be specific.
- Do not stage files with `git add -A` or `git add .` — stage by name.
- Do not categorize something as `FIX` when the reviewer's point is subjective and you disagree. Use `REPLY` and explain the reasoning.
- Do not proceed on `DEFER` items without asking the user first.
