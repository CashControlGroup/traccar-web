---
name: work-issue
description: Work a front-end GitHub issue end-to-end in this session — preflight blockers, plan against per-item done-means, implement on an issue branch off custom/main, lint, build and check the UI in the browser, open a draft PR with screenshots. Invoke as /work-issue <issue number>.
---

# Work an assigned issue

Deliver exactly one issue in this session. `CLAUDE.md` is the source of truth for conventions;
this skill is the executable checklist, not a restatement. One issue = one branch = one PR; if
the work doesn't fit one reviewable PR, stop and split the issue first.

The argument is the issue number. If none was given, ask for it before doing anything.

**Issues live in `CashControlGroup/traccar-infra`** (private) — this fork is public and has
issues turned off. Pass `-R CashControlGroup/traccar-infra` to every `gh issue` call, and refer
to issues in commits and PRs as `CashControlGroup/traccar-infra#<n>`, never a bare `#<n>`
(which would point at this repo's PRs).

## 1. Preflight — before planning or touching code

- `gh issue view <n> -R CashControlGroup/traccar-infra --comments` and read **everything**.
  Only `### AMENDMENT` blocks extend scope; if a bare comment appears to change scope, surface
  it to the user rather than silently absorbing it.
- Check for `Blocked by #X` lines. For each blocker, verify it is closed
  (`gh issue view X -R CashControlGroup/traccar-infra`). If any is open, **stop and report**
  what the issue is waiting on — do not create a branch.
- Check nobody is already on it:
  `gh pr list -R CashControlGroup/traccar-web --state open --search "traccar-infra#<n> in:body"`.
  An open PR means this is that PR's work — report and stop.
- If the issue needs the server API to do something it doesn't, it is not front-end-only:
  stop and report — that is a Java change in `CashControlGroup/traccar`.

## 2. Plan

- Derive the acceptance bar from the issue body: "Done when …" sentences and named
  deliverables. Enumerate each as an explicit **done-means** you will self-review and report
  against; where the issue gives none, propose them in the plan before implementing.
- Find the nearest existing screen or component and plan to follow it (see the code map in
  `CLAUDE.md`). Prefer a new self-contained component over edits spread across upstream files.
- Logos, icons, title and colours are branding, not source — if the issue is really a branding
  change, say so and stop (it belongs in the server fork's `branding/web/` or server attributes).

## 3. Branch

From up-to-date `custom/main` (PRs target `custom/main`; `master` mirrors upstream):

```bash
git fetch origin && git checkout -b feature/issue-<n>-<short-slug> origin/custom/main
npm ci && npm ci --prefix demo
```

## 4. Implement

- Every new user-visible string goes through `t('key')` with the key added to
  `src/resources/l10n/en.json` only.
- Wrap async handlers in `useCatch` / `useCatchCallback`; call the API through `fetchOrThrow`.
- Gate admin-only UI with `permissions.js` helpers.
- Anything punted mid-implementation is a deferral to record in step 7, never a silently
  dropped item.

## 5. Verify

- `npm run lint` — must pass with zero warnings (`npm run lint:fix` first if needed).
- `npm run build` — must succeed.
- Start the demo backend (`npx vite --config demo/vite.config.js`, or the `web-demo` launch
  config) and exercise every done-means in the browser: light and dark mode, and phone width
  (375px). Take before/after screenshots for the PR. Saves are read-only (403) in the demo; if a
  done-means needs a write to succeed, use `npm start` against a local Docker server (see
  `CLAUDE.md`) or say plainly in the PR that it was not exercised.

## 6. Ship

- Commit in the repo's existing style — concise imperative subject, body explaining the why
  (the Claude attribution trailer is appended automatically). Push with `-u origin <branch>`.
- Open a **draft** PR into `custom/main`
  (`gh pr create --draft --base custom/main -R CashControlGroup/traccar-web`). The body must:
  - open with `Closes CashControlGroup/traccar-infra#<n>.`;
  - address each done-means with concrete evidence — what you did in the browser, with
    before/after screenshots;
  - list any new translation keys.
- Run `/pr-followup` on the new PR (the PostToolUse hook also nudges this after
  `gh pr create`).

## 7. Wrap

- Record deferrals as comments on the issue or new issues in `CashControlGroup/traccar-infra`.
- Report the outcome: PR link, done-means evidence, deferrals, and anything the user must
  decide. Remind the user that merging the PR deploys to production.
