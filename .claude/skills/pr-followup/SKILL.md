---
name: pr-followup
description: Collect and triage automated reviewer feedback (Copilot, CI) on a just-created or updated traccar-web PR, apply agreed fixes, and reply to review comments. Invoked automatically by the PostToolUse hook after `gh pr create`; can also be run manually with a PR number or URL as the argument.
---

# PR Followup

Triage automated review feedback on a traccar-web pull request and act on it. The argument (or
the hook context) identifies the PR; when only a URL is available, extract `<owner>/<repo>` and
the PR number from it. Default repo: `CashControlGroup/traccar-web`.

## Steps

1. **Request the Copilot review explicitly** — auto-review does not fire for every PR:
   ```bash
   gh api repos/<owner>/<repo>/pulls/<n>/requested_reviewers -f 'reviewers[]=copilot-pull-request-reviewer[bot]'
   ```

2. **Wait for the review.** It typically lands 1–3 minutes after the request. Poll up to
   ~4 minutes (use a Monitor, not chained sleeps), counting only Copilot's reviews. On a re-run
   after pushing fixes, record the count before requesting and wait for it to increase:
   ```bash
   gh api repos/<owner>/<repo>/pulls/<n>/reviews \
     --jq '[.[] | select(.user.login == "copilot-pull-request-reviewer[bot]")] | length'
   ```
   If no new Copilot review arrives in the window, report that and carry on with CI.

3. **Fetch all feedback** — review comments plus CI:
   ```bash
   gh api repos/<owner>/<repo>/pulls/<n>/reviews   --jq '.[] | {user: .user.login, state, body}'
   gh api repos/<owner>/<repo>/pulls/<n>/comments  --jq '.[] | {id, user: .user.login, path, line, body}'
   gh pr checks <n> -R <owner>/<repo>
   ```
   The `custom-ci` workflow runs lint and build on every PR into `custom/main`, drafts
   included, so a failing check is real signal.

4. **Triage each finding** into one of:
   - **Fix** — the finding is correct: apply the change on the PR branch, following
     `CLAUDE.md` (translations via `t()`, existing components, upstream patterns).
   - **Decline with rationale** — it conflicts with `CLAUDE.md` or upstream's style (for
     example, a suggestion to restructure upstream code we deliberately leave alone to keep
     merges cheap): reply on the comment with the reason.
   - **Out of scope** — real but belongs elsewhere: reply saying so and flag it for a new issue
     in `CashControlGroup/traccar-infra` rather than widening the PR.

5. **Verify** — `npm run lint` and `npm run build`, and re-check any changed UI in the browser
   against the demo backend. All green before pushing.

6. **Push** the fixes to the PR branch as one commit summarising which findings it addresses.

7. **Reply to each review comment** with what was done:
   ```bash
   gh api repos/<owner>/<repo>/pulls/<n>/comments/<comment_id>/replies -f body='<reply>'
   ```

8. **Mark a draft ready once triage is settled** (`gh pr ready <n> -R <owner>/<repo>`), unless
   the user asked for it to stay draft.

9. **Summarise** in chat: findings fixed, findings declined (and why), anything flagged out of
   scope — and that merging deploys to production.

## Guard rails

- Never force-push; always additive commits on the PR branch.
- Never merge the PR yourself; merging ships to production and is the user's call.
- If the PR branch has diverged from the working tree, check it out first
  (`gh pr checkout <n> -R <owner>/<repo>`).
