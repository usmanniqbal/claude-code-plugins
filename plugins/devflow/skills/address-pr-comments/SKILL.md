---
name: address-pr-comments
description: Use when the user asks to address PR feedback, respond to review comments, fix PR comments, resolve review threads, or fix failing CI on a PR they've authored
---

# address-pr-comments

Triage unresolved review threads, failing CI checks, and DeepSource issues on a PR. For each item, either fix the code or draft a reply, preview the plan, and only act on explicit approval.

**REQUIRED:** Read `../../CONVENTIONS.md` first. It defines tool access, anchored references, writing style, fixup-commit policy, and which CI checks to skip as expected-fail.

## Steps

1. **Resolve the repo and PR number** per `CONVENTIONS.md` (parse a PR URL if given, else `gh repo view --json nameWithOwner`, else ask). Cache `<owner>/<repo>` and `<n>`. Then fetch PR state:
   - `gh pr view <n> --repo <owner>/<repo> --json title,body,number,headRefName,headRefOid,baseRefName,url,author,statusCheckRollup,comments,reviews`
   - Unresolved review threads via GraphQL (thread IDs are required to reply/resolve):

     ```bash
     gh api graphql -F owner=<owner> -F name=<repo> -F number=<n> -f query='
       query($owner:String!,$name:String!,$number:Int!){
         repository(owner:$owner,name:$name){
           pullRequest(number:$number){
             reviewThreads(first:100){
               nodes{ id isResolved isOutdated comments(first:50){ nodes{ id databaseId path line body author{login} } } }
             }
           }
         }
       }'
     ```

2. **Extract the Jira ticket** from the PR title, body, or branch name (`EXP-\d+`). Ask the user if none can be found.

3. **Check PR ownership.** Compare `gh api user --jq .login` to the PR author. If different, ask:

   > "This PR was authored by @<author>, not you. Have they handed it off to you? Confirm to proceed."

   Wait for explicit confirmation. Skip silently if the user is the author.

4. **Suggest a session rename** as the next line of your response, on its own line:

   ```
   /rename Addressing - <TICKET> - <PR title>
   ```

   `<TICKET>` = ticket key (never the PR number). `<PR title>` = exact title from `gh pr view --json title`.

5. **Build the triage list.** One entry per: unresolved review thread, top-level PR comment without an author response, failing check (after filtering the expected-fail checks listed in CONVENTIONS.md).

6. **DeepSource failures**: don't read the check log. Instead:
   - `deepsource auth status`. If unauth'd, stop and tell the user to run `deepsource auth login` (install: `curl -fsSL https://cli.deepsource.com/install | sh`).
   - `deepsource issues --pr <n> --output json`. Filter with `--severity`, `--analyzer`, `--category`, `--path` if the list is long.
   - Add each issue as `FIX (DeepSource)` to the triage list.
   - After pushing fixes, re-analysis runs automatically; verify with `gh pr checks <n>`.

7. **Categorize each item**:
   - `FIX (code)`: concrete code change.
   - `FIX (CI)`: failing check with a clear cause; confirm via `gh run view <run-id> --log-failed`.
   - `FIX (DeepSource)`: from `deepsource issues`.
   - `REPLY`: question, subjective preference, or already addressed.
   - `DEFER`: ambiguous; ask the user before acting.

   **Tag the reviewer type per thread.** Check each thread's author login. Bot accounts (login ends `[bot]`, or matches known AI review bots: `augmentcode`, `copilot`, `github-actions`, `deepsource`, `codecov`, similar) get the **AI flow** in step 8. Everyone else is **human**.

8. **Draft per item.** For `FIX`, locate the code and plan the edit (don't edit yet). For `REPLY`, draft the exact text. Whether you draft a reply at all depends on the reviewer type:

   **Human reviewer threads.** Every `FIX` and `REPLY` gets a draft reply describing what changed in human terms (per `CONVENTIONS.md` "Writing PR replies and comments"), not a commit anchor. Every reply waits for explicit approval before posting; code fixes can be applied and pushed on the user's "yes," but the reply itself stays human-in-the-loop.

   **AI reviewer threads.** No human is waiting; close the loop with reactions instead of noise:
   - `FIX` where the fix matches what the bot proposed → **react 👍 on the bot's original comment + resolve the thread. No reply.** A "Fixed!" comment is noise when the fix is identical to the suggestion.
   - `FIX` where the fix differs from the proposal, OR the bot asked a question → 👍 + resolve **PLUS** a concise reply explaining the diff or answering.
   - `REPLY` where you disagree with the bot's suggestion → reply with the disagreement AND react 👎 on the bot's original comment. **Do NOT resolve the thread** — leave it for a human reviewer to weigh in.

   `FIX (DeepSource)` items don't need a thread reply or reaction — they aren't review threads; the green re-analysis check is the signal.

9. **Preview the full plan** in one message:

   ```
   Item 1, thread on src/foo.ts:42 (reviewer @alice, human)
     Category: FIX (code)
     Edit:    src/foo.ts:42, tighten null check on user.email
     Reply:   "Good catch, tightened the null check so the falsy case no longer reaches the formatter."

   Item 2, thread on src/bar.ts:18 (reviewer @augmentcode-ai[bot], AI)
     Category: FIX (code) — fix matches proposal
     Edit:    src/bar.ts:18, use Array.from instead of spread
     Action:  👍 + resolve, no reply

   Item 3, thread on src/baz.ts:7 (reviewer @copilot[bot], AI)
     Category: REPLY — disagree with bot's suggestion
     Reply:   "Keeping the explicit type here so the inferred union doesn't widen at the call sites."
     Action:  reply + 👎, leave thread open for human

   Item 4, CI: lint
     Category: FIX (CI)
     Cause:   unused import
     Edit:    src/qux.ts:3, remove unused `foo`

   Skipped (expected-fail on fixups): Commit / Autosquash, Commit message lint

   Summary: 3 fixes, 1 fixup commit targeting <sha>, 2 replies, 1 reaction, 1 thread resolve.
   ```

   Wait for explicit approval. The user may approve all, a subset, or request edits. No action on silence.

10. **Apply approved fixes**: edit the code, show the diff before staging, stage by file name (no `-A`/`.`), `git commit --fixup <original-sha>` per relevant main commit, push.

11. **Post approved replies, react, and resolve threads.**

    Reply to a thread:

    ```bash
    gh api graphql -F threadId=<id> -F body="..." -f query='
      mutation($threadId:ID!,$body:String!){
        addPullRequestReviewThreadReply(input:{pullRequestReviewThreadId:$threadId,body:$body}){ comment{ id url } }
      }'
    ```

    React 👍 / 👎 to an AI reviewer's original comment (use `content=+1` or `content=-1`). The `<databaseId>` is the comment's REST id, returned by the `reviewThreads` GraphQL query as `comments.nodes[].databaseId` — NOT the node `id`:

    ```bash
    gh api -X POST repos/<owner>/<repo>/pulls/comments/<databaseId>/reactions -f content=+1
    ```

    Resolve once the fixup is pushed (skip this for 👎-disagreement threads on AI reviewers — leave them open for a human):

    ```bash
    gh api graphql -F threadId=<id> -f query='
      mutation($threadId:ID!){ resolveReviewThread(input:{threadId:$threadId}){ thread{ id isResolved } } }'
    ```

    Top-level PR comments: `gh pr comment <n> --body "..."`. CI/DeepSource-only fixes need no reply; the green check is the signal.

12. **Report back**: fixup commits with SHAs and URLs, threads replied/resolved, new CI status (`gh pr checks <n>`).

## Do not

- Push code or post replies without preview and explicit approval.
- Proceed past step 3 on someone else's PR without an explicit handoff.
- Amend or rebase; fixup commits only.
- Read DeepSource issues from the check run log; use the CLI.
- Resolve a thread until the fix is pushed.
- Mass-reply "fixed!" to threads you didn't actually fix.
- Reply "Fixed!" to an AI reviewer when the fix matched their proposal — react 👍 + resolve instead. The reaction is the canonical "addressed" signal for these bots; a reply is noise.
- Resolve an AI reviewer thread you disagreed with — react 👎 + reply with your reasoning and leave the thread open for a human to weigh in.
- Lead a reply with a commit SHA. Describe the change in plain language; anchor the commit only when the SHA itself is genuinely useful.
- Categorize a subjective disagreement as `FIX`; use `REPLY`.
- Proceed on `DEFER` items without asking.
