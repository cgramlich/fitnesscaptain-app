# FitnessCaptain App

Single-file React PWA for fitness and exercise logging (Forever Apps
portfolio). Copied from the HomeCaptain shell (the freshest clean extract of
the MenuCaptain/Tracker template) and reskinned; auth patterns (forgot
password, recovery, password-reveal eyeball) ported from Tracker.

## Architecture

- `index.html` - the whole app: theme tokens, store, auth, screens. React via
  CDN + Babel standalone; compiled in the browser, no build step.
- `sw.js` - offline service worker: network-first app shell, cache-first
  vendored assets, network-first data cache. Subpath-safe for GitHub Pages.
- `check.js` / `npm run check` - Babel-in-Node compile gate. Run before every
  deploy.
- Deliberately NO web manifest (template decision): Add to Home Screen then
  uses the current page URL as the launch URL.

## Versioning (bump on EVERY deploy)

- `APP_VERSION` (semver, user-facing) + `BUILD` (YYYY-MM-DD.N, what the
  updater compares) in index.html
- `VERSION` in sw.js in lockstep
The updater checks on launch, on resume, and every 20 minutes.

## Data model (NAMES ARE FOREVER)

- localStorage namespace `fitnesscaptain:`; Supabase auth storageKey
  `fitnesscaptain-auth`. Never rename after launch.
- Collections `exercises`, `workouts`, `routines` (arrays of records) + `meta`
  (per-user singleton holding `cfg`). Whole-collection GET/PUT via the
  backend's `/api/collection/{name}`.
- Store: optimistic local writes + per-collection dirty flags; dirty
  collections are skipped on pull and re-pushed on flush/online/sign-in.
  Sign-out wipes the local cache.

## Auth (Supabase, browser side = auth only)

Sign in / sign up (email confirm flow with the pending-email trap handled),
forgot password -> reset email -> PASSWORD_RECOVERY -> SetPassword screen.
EVERY password field uses the shared `PwInput` reveal-eyeball component
(portfolio standard). Raw vendor errors are translated to human copy.

## Config placeholders

`SUPABASE_URL` / `SUPABASE_ANON_KEY` / `API_BASE` at the top of the script
block. All public-safe. `CONFIGURED` gates the sign-in button until set.

## Build order (from the kickoff doc)

Step 1 (this shell) done. Next: 2 exercise library (seeded ~50), 3 fast
workout logging UX (the product), 4 routines via the Templates pattern,
5 progress views, 6 AI (progress_summary / routine_suggest / explain_exercise
via `aiRelay`), phase 2 metrics/goals.

## Working practices

- ASCII-only in code and logs, no emoji. Log via `log(tag, msg)` - it feeds
  the Settings diagnostics screen.
- Keep this file + the docs in `Dropbox\My AI\CG Apps\Health Tracker\` current
  in the SAME session that changes code (doc-currency rule).
- Log portable wins in `CG Apps\Forever Apps\PORTABLE-IMPROVEMENTS.md`.
