# FitnessCaptain App

Single-file React PWA for fitness and exercise logging (Forever Apps
portfolio). Copied from the HomeCaptain shell (the freshest clean extract of
the MenuCaptain/Tracker template) and reskinned; auth patterns (forgot
password, recovery, password-reveal eyeball) ported from Tracker.

## Doc currency (Forever Apps starter spec section 5)

Update docs in the SAME session as the code change: this CLAUDE.md stays true to
the code, and the docs in `Dropbox\My AI\CG Apps\Health Tracker\` update when the
architecture changes. Write a DATED entry in
`C:\Users\cjgra\Dropbox\My AI\CG Apps\Health Tracker\Fitness Log\` for EVERY work
session. Never hardcode versions in docs; point at `APP_VERSION`/`BUILD` in
index.html and the backend's live `/health`.

## Code readability (Chris's directive 2026-08-10) - part of "done"

The code has to be readable by three people who were not in the room when it
was written: a human AUDITOR reconstructing what the system does, a REGULATOR
asking "show me where that rule is implemented", and a DEVELOPER JOINING COLD.
If any of them has to ask "why is this here?", the code has failed - whether or
not it works. This sits alongside "it compiles" and "it is deployed" in the
definition of done; it is not polish to add later.

- Comment the WHY and the RULE, never the WHAT. The test: a good comment is
  still true after someone rewrites the implementation. If it only narrates the
  current lines, delete it.
- State every business rule in plain English beside the code enforcing it, so a
  reviewer reads the rule and sees the code without inferring it from the logic.
  Say where the rule came from (policy, provider limit, a decision Chris made)
  and date it.
- Record the road not taken on non-obvious decisions - what was rejected and
  why - so a later session doesn't "fix" a deliberate choice.
- Flag anything surprising with a WARNING comment where someone would trip over
  it: load-bearing quirks, footguns, things that fail silently.
- Open every file with an orientation block (what it is, what it owns, what it
  deliberately does not do) and split long files into labelled banner sections
  (`STORE`, `AUTH`, `LOGGING`, `AI`). Matters most in index.html, which runs to
  thousands of lines by design.
- Names are the documentation. Explicit over clever, always - don't golf.
- Delete dead code, never comment it out. Git remembers.

The opposite failure is just as bad: NOT a comment per line, NOT the code
restated in English, NOT ceremonial docblocks, NOT commit-message content in
comments (who changed what and when is git's job). Noise buries signal and
teaches the reader to skip comments, including the one that mattered.
Restructure unclear code instead of apologizing for it in a comment.

Full standard (the authority, read it):
`C:\Users\cjgra\Dropbox\My AI\CG Apps\Forever Apps\CODE-READABILITY-STANDARD.md`

## Pre-push audit gate (never bypass)

`git push` runs the shared checker
`C:\Users\cjgra\portfolio-audit\portfolio_audit.py` through
`.git\hooks\pre-push`. Backend checks BLOCK the push; frontend checks are
advisory (print-only) for now. Run it ad hoc any time with `--all`.

It exists because these specific failures actually happened:

- a model that was routed but had no price entry took PriorityCaptain's AI
  relay fully offline
- a duplicate dict key made THIS app's backend meter Sonnet 5 at Haiku's rate,
  so the budget breaker sailed past roughly 3x the real spend
- a typo in a Railway numeric variable crash-loops a backend at import

`--no-verify` is NOT an acceptable workaround - it is already against Chris's
standing rule. If the gate is wrong, fix the checker; don't push around it.

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
