---
name: create-pr
description: Use when the user asks to open a pull request, create a PR, or push a feature branch and surface it for review
---

# create-pr

Open a PR that matches the repo's conventions and won't bounce on missing references or wrong format.

**REQUIRED:** Read `../../CONVENTIONS.md` first. It defines tool access, writing style, and commit conventions.

## Steps

1. **Confirm you're in the right repo's clone.** `create-pr` runs against the cwd's git repo. If `gh repo view --json nameWithOwner` doesn't match the user's intent (or fails), stop and ask which repo before continuing.

2. **Read the repo's PR template** at `.github/pull_request_template.md`. Match its sections exactly (common ones: Type, Ticket, Packages, Details, Verification Steps, Blockers, Followup, Checklist). If no template exists, fall back to a concise Summary + Test Plan.

3. **Resolve `<base>`** from the default branch (per `CONVENTIONS.md`): `gh repo view --json defaultBranchRef -q '.defaultBranchRef.name'`. Don't assume `main` or `master`.

4. **Gather branch context** in parallel:
   - `git status` (no `-uall`)
   - `git diff` (staged + unstaged)
   - `git log <base>..HEAD` and `git diff <base>...HEAD` so you see every commit, not just the latest
   - Branch tracking state

5. **Verify referenced configs.** If the diff touches Dependabot, CODEOWNERS, workflows, tsconfig paths, or `package.json` scripts, confirm every referenced package, path, or file actually exists. Misspellings and stale references are the top reason PRs bounce.

6. **Draft title and body**:
   - Title under 70 chars, Conventional Commits style if the repo uses it.
   - Body follows the template.
   - Verification Steps must be realistic: ask reviewers to verify the implementation, not to wait for CI.
   - Include the Jira ticket reference (e.g., `EXP-XXXXX`) in the title or a dedicated Ticket field.

7. **Push the branch** with `-u` if it isn't tracking a remote.

8. **Create the PR** via `gh pr create` with a HEREDOC body for correct formatting.

9. **Return the PR URL.**

## Do not

- Invent template sections the repo doesn't use.
- Skip dependency verification on config-touching PRs.
- Ask reviewers to "wait for CI to pass" as a verification step.
