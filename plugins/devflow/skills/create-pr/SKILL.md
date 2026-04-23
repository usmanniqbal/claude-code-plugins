---
name: create-pr
description: Create a pull request following the repo's PR template, with all config references verified against the actual codebase. Use when the user asks to open/create a PR.
---

# create-pr

Create a pull request that matches the repo's conventions and won't fail review due to missing references or wrong format.

## Steps

1. **Read the repo's PR template** at `.github/pull_request_template.md`. Match its structure exactly (common sections: Type, Ticket, Packages table, Details, Verification Steps, Blockers, Followup, Checklist). If no template exists, fall back to a concise Summary + Test Plan.

2. **Gather branch context** in parallel:
   - `git status` (no `-uall`)
   - `git diff` (staged + unstaged)
   - `git log <base>..HEAD` and `git diff <base>...HEAD` to see every commit in the PR, not just the latest
   - Current branch tracking state

3. **Verify all referenced configs/dependencies**. If the diff touches config files (Dependabot, CODEOWNERS, workflows, tsconfig paths, package.json scripts, etc.), confirm every referenced package/path/file actually exists in the repo. A PR that references a deleted package or misspelled path will bounce.

4. **Draft title and body**:
   - Title under 70 chars, follows Conventional Commits style if the repo uses it
   - Body follows the template sections
   - **Verification Steps must be realistic** — ask reviewers to verify the implementation, not to wait for CI or external systems
   - Include the Jira ticket reference (e.g., `EXP-XXXXX`) in the title or a dedicated Ticket field

5. **Push the branch** with `-u` if it isn't tracking a remote.

6. **Create the PR** using `gh pr create` with a HEREDOC body for correct formatting.

7. **Return the PR URL** so the user can open it.

## Do not

- Do not invent template sections the repo doesn't use.
- Do not skip the dependency verification step on config-touching PRs.
- Do not ask reviewers to "wait for CI to pass" as a verification step.
