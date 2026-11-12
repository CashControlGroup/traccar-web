# CLAUDE.md

Guidance for Claude Code sessions in `CashControlGroup/traccar-web`.

## What this is

Our fork of [traccar-web](https://github.com/traccar/traccar-web), the React front-end of the
Traccar GPS tracking server, deployed white-labelled as **Tracking Shop**. It is a Vite single-page
app: React 19, MUI 9, Redux Toolkit, MapLibre. It talks to the Traccar server's REST API
(`/api/*`) and WebSocket (`/api/socket`).

Related repos, each with its own CLAUDE.md:

- `CashControlGroup/traccar` — the Java server. This repo is its `traccar-web` submodule.
- `CashControlGroup/traccar-infra` (private) — Azure infra, deploy workflow, and the **issue
  tracker** for this work (the forks are public and have issues turned off).

> **Global standards do not apply here.** Jamie's global standards (C#/.NET, DDD layering,
> Blazor/MAUI) describe our greenfield work. This is upstream's React code: match the patterns
> in the file you are editing, and keep our changes small so upstream merges stay cheap.

## Branches

- `master` mirrors `traccar/traccar-web`. Never commit to it.
- `custom/main` is what we build and deploy. **Base all work on `custom/main`; PRs target
  `custom/main`.** Feature PRs may be squash-merged. Upstream-sync PRs are merged with a merge
  commit, never squashed.
- Merging into `custom/main` ships to production: `ship-to-server` moves the server fork's
  submodule to the merged commit, which rebuilds and deploys (see the server repo's CLAUDE.md).

## Run it locally

Node 22 (`.npmrc` sets `legacy-peer-deps`; always `npm ci`, never `npm install` for setup).

```
npm ci
npm ci --prefix demo          # once, for the demo backend
```

- **Demo backend, no server needed (default for UI work):**
  `npx vite --config demo/vite.config.js` → http://localhost:3000, auto-logged-in to a simulated
  fleet with live positions, reports and geofences. It is **read-only**: any POST/PUT/DELETE
  returns 403, so screens that save data cannot be exercised end to end.
- **Real backend:** `npm start` proxies `/api` to `localhost:8082`. Run a local server with
  Docker: `docker run -d --name traccar -p 8082:8082 traccar/traccar:6-alpine`, then register
  the first user (who becomes admin) on the login page. Never point the dev server at the
  production domain.

`.claude/launch.json` defines both (`web-demo`, `web-live`) for the desktop app's preview pane.

## Verify before every PR

- `npm run lint` — ESLint + Prettier, **zero warnings allowed** (`npm run lint:fix` repairs most).
  `.gitattributes` keeps every file LF; if lint floods with ``Delete `␍` `` on Windows, the
  checkout predates it — re-checkout with `git rm -r -q --cached . && git reset -q --hard`
  (on a clean tree only).
- `npm run build` — must succeed.
- There is no unit-test suite. Check UI changes in the browser against the demo backend, in
  light and dark mode and at phone width, and include before/after screenshots in the PR.

## Code map

- `src/Navigation.jsx` — all routes. `src/main/` live map + device list; `src/reports/`;
  `src/settings/` (edit pages); `src/login/`; `src/map/` (MapLibre layers and controls);
  `src/other/` (position, replay, etc.).
- `src/store/` — Redux Toolkit slices (`session`, `devices`, `events`, …). `SocketController.jsx`
  feeds live updates into them.
- `src/common/components/` — shared building blocks (`PageLayout`, `SelectField`, `LinkField`,
  `RemoveDialog`, …). In `src/settings/`, list pages (`*sPage.jsx`, with `CollectionActions`)
  and edit pages (`*Page.jsx` on `components/EditItemView.jsx`) are the pattern for CRUD
  screens. Reuse these before writing new ones.
- `src/common/util/` — `fetchOrThrow` (all API calls), `formatter.js`, `preferences.js`
  (`useAttributePreference`), `permissions.js` (`useAdministrator`, `useRestriction`, …).
- `src/reactHelper.js` — `useCatch` / `useCatchCallback` / `useAsyncTask`; wrap async handlers
  with these so errors surface in the standard error snackbar.
- `src/common/theme/` — MUI theme; colours come from server attributes (`colorPrimary`, …).

## Conventions

- **Every user-visible string is translated.** Use `const t = useTranslation();` and
  `t('keyName')`; add new keys to `src/resources/l10n/en.json` only (other locales are filled by
  upstream's Transifex sync and fall back to English).
- Style with `makeStyles` (from `tss-react`) or MUI `sx`, as the surrounding file does; no new
  CSS files or component libraries.
- Gate admin-only UI with the helpers in `permissions.js`, not ad-hoc checks on `session.user`.
- **Branding is not done in source.** Logos, favicon and PWA icons are overlaid at Docker build
  time from `branding/web/` in the server fork; title and colours come from server attributes.
  Changing a file name under `public/` breaks that overlay — check with Jamie first.
- Prefer small, self-contained changes (a new component or page) over edits scattered through
  upstream files; every upstream line we change is a future merge conflict. If a change needs
  the server API to do something it doesn't, stop and raise it — that is a Java change in the
  server fork.
