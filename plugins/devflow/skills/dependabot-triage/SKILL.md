---
name: dependabot-triage
description: Use when triaging open Dependabot PRs in a repo — deciding which dependency bumps are safe to merge, why a green PR is blocked, whether a failing check traces to the bump, or whether already-merged bumps were safe. Also use when an on-call rotation asks for a Dependabot backlog sweep.
---

# dependabot-triage

Research the open Dependabot PRs in a repo, categorize each by merge safety, and recommend an order. **Read-only on GitHub by default:** every merge, approval, thread resolution, comment, and `@dependabot` command waits for explicit per-PR approval from the user. You produce research and a recommendation; the user drives the writes.

**REQUIRED:** Read `../../CONVENTIONS.md` first. It defines repo and default-branch resolution, tool access (MCP > CLI > curl), anchored references, and writing style. Apply throughout.

This skill assumes a JS/TS repo on Yarn (`yarn.lock`); the lockfile-cascade guidance is Yarn-specific. The CI/branch-protection/changelog logic is general.

## Steps

1. **List open Dependabot PRs** (author `app/dependabot`), oldest first, with number and title. Resolve `<owner>/<repo>` and `<base>` (default branch) per `CONVENTIONS.md`.

   **Flag prior-rotation strays.** Scan the list for PRs that have been open notably longer than the current rotation window (e.g., months old when the rest are days old). They're usually leftovers from a prior on-call: either superseded by a newer Dependabot PR for the same dep, or genuinely held back for a reason that's no longer documented. Surface them separately at the top of the report with a "needs a decision" note — don't bury them in the main triage table.

2. **For each PR, gather:**
   - **CI status:** the full `statusCheckRollup` (which checks pass, fail, pending).
   - **`mergeable` + `mergeStateStatus`.** Re-poll if it returns `UNKNOWN` — GitHub computes it lazily.
   - **The actual version bumps:** for npm PRs from the `package.json` diff; for `github_actions` PRs from the workflow `uses:` diff. Note major vs minor/patch, and runtime dep vs dev/tooling dep.
   - **The release notes / changelog, not just the version delta.** Dependabot embeds them in the PR body (`gh pr view <n> --json body`). Read them and flag breaking changes, removals, deprecations, and any raised engine/runtime requirement. Version numbers alone don't tell you what changed. If the PR body's changelog is thin, or the version is newer than your own knowledge, fetch the upstream changelog / "upgrading to vN" guide directly — don't infer breaking changes from the version number or a CI log alone.
   - **For a GROUP PR** (title "bump the X group"), the diff mixes several packages at different bump levels: **categorize PER PACKAGE and let the WORST package set the tier.** A benign minor can hide a breaking major sibling (e.g. a `next` 16.1→16.2 minor bundled with an `eslint-config-next` 15→16 major that's the actual breaker).
   - **Raised engine requirement** (e.g. a major that now needs Node ≥22): cross-check against the repo's `.nvmrc` / CI `node-version-file`, and confirm the relevant check actually ran green on the NEW version (a green commit-lint/test job is live proof CI's runtime satisfies it). If there's no `engines` pin in `package.json`, flag local-dev risk for anyone on an older runtime.
   - **Unresolved review threads** (GraphQL `reviewThreads` → count `isResolved == false`, and read them). If branch protection requires thread resolution (step 3), an open thread BLOCKS merge even with green CI + approval — so a green PR sitting at `mergeStateStatus: BLOCKED` may be waiting on a thread, not just an approval. AI reviewers (augmentcode, Copilot) commonly leave these; read each one and judge whether it's a real blocker or a style nit. Resolving a thread is a write — it waits for the user's okay.

     **`isOutdated: true` ≠ resolved.** GitHub marks a thread "outdated" when the comment's line moved (the file changed underneath it); the thread is still `isResolved: false` and still gates merge under `required_review_thread_resolution: true`. Don't filter outdated threads out of the unresolved count.

3. **Read branch protection on `<base>`:** required approving reviews, required status checks, `strict` (up-to-date required?), `enforce_admins`. Also the repo's allowed merge methods.
   - If `gh api repos/<owner>/<repo>/branches/<base>/protection` returns 404 "Branch not protected", the repo enforces protection via **RULESETS**, not classic protection. Fall back to `gh api repos/<owner>/<repo>/rulesets`, then read each branch-targeting ruleset's detail (`.../rulesets/<id>`). Pull the same fields from the `pull_request` and `required_status_checks` rule params: `required_approving_review_count`, `required_status_checks[].context`, `strict_required_status_checks_policy` (= `strict`), `allowed_merge_methods`, and `required_review_thread_resolution`. `bypass_actors` is the enforce-admins equivalent — empty means it's enforced on everyone, admins included.

4. **Categorize** (judge CI against the REQUIRED checks from step 3 — a failing NON-required check like Knip or DeepSource is a note, not a blocker; don't drop a PR to ⛔ for it):
   - ✅ **Green CI + simple** (minor/patch, or dev-only) — safe to merge
   - 🟡 **Green CI but a major bump or a sensitive runtime dep** — needs a closer look
   - ⛔ **Failing CI** (confirm the failure is the bump's fault, not CI infra — see step 5) **or risky** (major framework upgrades, e.g. React/Capacitor/build tooling) — not safe as-is

5. **Don't trust green CI blindly.** For each green PR, check **staleness:** when the branch was last rebased, how many commits it's behind `<base>`, and whether those commits touch anything related to the bump. If `strict=false`, a stale-but-green PR merges WITHOUT re-running CI against current HEAD — call that out. Be honest that green tests prove the suite passes, not zero runtime behavior change for runtime deps.

   **Don't trust RED CI blindly either.** A failing REQUIRED check on a Dependabot PR is often environmental, not the bump's fault — diagnose before dropping it to ⛔. Read the failing job's annotations/log (`gh api repos/<owner>/<repo>/check-runs/<id>/annotations`) and find the actual error. The classic false-⛔ in private-registry monorepos: the Docker image / `Build` job fails at `yarn install --immutable` with `YN0041: ... Invalid authentication (as an anonymous user)` fetching private `@<scope>/*` packages — that's `NPM_TOKEN` missing in Dependabot's restricted secret context (Dependabot secrets are separate from Actions secrets), NOT the dependency. Confirm by checking the SAME job passes on `<base>`'s recent commits. When the failure is infra, give the PR its true tier (usually 🟡/✅ "benign, blocked by CI infra — fix the Dependabot secret, then rebase"), not ⛔. Reserve ⛔ for failures that trace to the upgraded package (a real test/lint/type error).

6. **Present a table:** PR #, package, version bump, type, CI state, clickable PR link (anchored per `CONVENTIONS.md`). Give a clear recommendation per tier, then stop — the user approves and merges themselves. In the recommendation, **separate the merge gates:** mark which green PRs need only approval vs. which ALSO need an unresolved review thread resolved first (from step 2). They're distinct gates and the user clears them differently.

## Merging several Dependabot PRs (lockfile cascade)

The npm PRs all touch `yarn.lock`:

- `github_actions` PRs touch only workflow YAML, not `yarn.lock` — they're independent of the lockfile churn and can merge in any order.
- After the first npm PR merges, the others may flip `MERGEABLE` → `CONFLICTING` on `yarn.lock`.
- Fix is `@dependabot rebase` (Dependabot regenerates the lockfile against current HEAD and force-pushes; CI re-runs). `@dependabot recreate` if rebase gets stuck.
- Suggest a merge order that minimizes rebase churn, and remind the user that merging one PR re-conflicts any not-yet-rebased sibling.
- If `dismiss_stale_reviews` is on (from step 3): approve a PR AFTER any needed rebase, not before — the rebase force-push dismisses the approval.
- **Superseded PRs:** if a PR is `CONFLICTING` and its "from" version no longer exists on `<base>` (e.g. a newer open PR bumps the same dep from a higher baseline), it's obsolete — recommend CLOSE, not rebase. Two open PRs for the same dep = the lower-baseline one is stale.
- Only post a `@dependabot rebase` comment when the user explicitly says to.

## If PRs were already merged (verify after the fact)

Sometimes the user merged some PRs before (or without) triage and just wants a safety check. For each merged PR / `<base>` HEAD:

- Pull the merge commit's checks: `gh api repos/<owner>/<repo>/commits/<sha>/check-runs`.
- Confirm the CORRECTNESS checks passed (tests, lint, translations). Those are the signal for "was this safe".
- Treat still-pending DEPLOY/BUILD checks (Docker `base-build` images, Stage/deploy pipelines) as orthogonal to merge safety — report them as "deploy pipeline, not a correctness risk", not as a problem.
- A stale PR merged without rebase under `strict=false` is validated by THIS post-merge run on `<base>` — wait for it to finish before giving an all-clear.

## Recency: read CI state, not timestamps

Don't infer how long ago something merged by comparing commit timestamps against today's date (there's no reliable wall clock). Use CI run STATE: a run still `in_progress` means the merge is recent (these finish in minutes, not hours); only `completed` conclusions are trustworthy. Re-poll pending runs instead of assuming they're stuck or stale.

## Do not

- Merge, approve, resolve a thread, or post any `@dependabot` comment without explicit per-PR approval.
- Call a bump "safe" from the version delta alone — read the changelog first.
- Drop a PR to ⛔ for a failing NON-required check, or for a CI-infra failure that doesn't trace to the package.
- Categorize a group PR by its mildest package; the worst package sets the tier.
- Approve before a needed rebase when `dismiss_stale_reviews` is on.
- Infer merge recency from timestamps; read CI run state instead.
