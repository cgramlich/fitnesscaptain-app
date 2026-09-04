# FitnessCaptain — HANDOFF

**State as of 2026-09-04.** App `v0.125.0`, backend `v0.29.0`, both live.
Written to the portfolio `DOCUMENTATION-STANDARD.md` (2026-08-24). Authoritative: where this and
`BRIEFING.md` disagree, **this file is right**.

---

## What this is

A workout logger. Chris trains across several gyms — a home setup, a club, hotel gyms while
travelling — and the app's whole reason for existing is that **a number only means something in
the room it was set in**. 100 lb on one cable stack is not 100 lb on another, so sessions, PRs
and machine labels all carry their gym.

It is a Forever App: parity build, not a store app. Personal use, built for one person, on the
portfolio's shared plumbing (auth, sync, offline shell, AI relay).

---

## Current state

| | | |
|---|---|---|
| App | `v0.125.0` | https://fitnesscaptain.com — GitHub Pages, repo `cgramlich/fitnesscaptain-app` |
| Backend | `v0.29.0` | Railway, repo `cgramlich/fitnesscaptain-backend` |
| Share links | live | `go.fitnesscaptain.com/g/{token}` → server-rendered `gym.html` |
| Data | Supabase | Postgres JSONB collections + a private Storage bucket |

Both repos are committed and pushed clean. Nothing is sitting uncommitted.

**Verify the live versions before trusting anything above:**

```bash
curl -s https://fitnesscaptain-backend-production.up.railway.app/health
```

That also reports `ai_last_call` — see Traps.

---

## The map

**`fitnesscaptain-app`** — single-file PWA. No build step; Babel compiles in the browser.

| Path | Owns |
|---|---|
| `index.html` | The entire app. ~10,460 lines, one `<script type="text/babel">` block |
| `sw.js` | Offline shell. `VERSION` must equal `APP_VERSION` |
| `check.js` | **Pre-flight gate.** Run before every push |
| `exercise-media.json` | 873-exercise reference index (names, equipment, illustration folders) |
| `gym.html` | Public share page for a gym |
| `welcome.html`, `privacy.html`, `terms.html`, `support.html` | Static, linked from legal/support |

**`fitnesscaptain-backend`** — FastAPI on Railway.

| Path | Owns |
|---|---|
| `main.py` | Everything: collections, AI relay, metering, Places, photos, shares |
| `community_gyms.sql`, `gym_shares.sql` | Schema. **Already run** in Supabase |

Endpoints group as: `/api/collection/{name}` (the sync spine), `/api/ai/relay`, `/api/places/*`,
`/api/progress/photo*`, `/api/share/*` plus `/g/{token}`, `/api/community/*`, `/api/account`,
`/health`, `/api/ai/models`.

**Railway variables** (names only — values live in Railway, never in a repo or a chat):
`ANTHROPIC_API_KEY`, `SUPABASE_URL`, `SUPABASE_SERVICE_ROLE_KEY`, `GOOGLE_PLACES_API_KEY`,
`YOUTUBE_API_KEY`, `PUBLIC_BASE_URL`, `SHARE_DEST_BASE`, `DEBUG_KEY`, `AI_MONTHLY_BUDGET_USD`,
`AI_MONTHLY_CALL_CAP`, `PLACES_MONTHLY_CAP`, `VIDEO_DAILY_CAP`, `AI_MODEL_DEFAULT`,
`AI_MODEL_<TASK>`.

---

## How to run, build, deploy

There is no build. Editing `index.html` **is** the build.

```bash
cd /c/Users/cjgra/fitnesscaptain-app && node check.js
```

`check.js` is not optional. It compiles the Babel block with the same preset the browser uses,
and enforces two tripwires that exist because both failures actually happened — see Traps.

Deploy the app (GitHub Pages picks it up in a minute or two):

```bash
cd /c/Users/cjgra/fitnesscaptain-app && git add -A && git commit && git push origin main
```

Deploy the backend (Railway redeploys on push, and on any variable change):

```bash
cd /c/Users/cjgra/fitnesscaptain-backend && python -c "import ast;ast.parse(open('main.py',encoding='utf-8').read())" && git add -A && git commit && git push origin main
```

**Version ritual, every app release — all three, or the updater lies:** `APP_VERSION` in
`index.html`, `BUILD` in `index.html`, `VERSION` in `sw.js`.

**Order matters when a change spans both repos.** If the app will send the backend something
new, **deploy the backend first and confirm `/health` reports the new version**, then ship the
app. The scan image cap is the live example: the relay rejects an over-cap request outright
rather than truncating, so the other order breaks every scan until Railway catches up.

**Testing is the harness.** Copy `index.html`, replace the Supabase `<script>` with a stub that
fakes `window.supabase` and `window.fetch`, disable the service worker, serve it, drive it with
the browser tools. Write the harness builder as a **`.js` file** — heredocs mangle escapes, and a
`\b` silently becoming a backspace has cost real time here more than once. (Writing *this* file
hit the same trap.)

---

## Decisions, dated, with the road not taken

**Cross-gym comparability is LOCAL, not global (2026-08-10).** No usable open catalogue of gym
machines exists; manufacturer lists are proprietary and rot. Rejected: building or licensing an
equipment database. Chosen: the app already knows which gym you are in, so sessions carry their
gym and a warning fires for machine/smith/cable but **not** barbell/dumbbell — 45 lb of plate
weighs 45 lb everywhere. Do not reopen this as "we should add a machine database".

**Machine names and photos are keyed by gym, stored on the exercise (2026-08-10 / 08-11).** "On
the IM2000" is a fact about a room. Per-exercise-only storage would carry it into every other
building.

**The name outranks the equipment tag (2026-08-15).** The reference DB files back extensions as
`body only`, which imports as `bodyweight`, so a "Seated Back Extension **Machine**" was classed
as needing no apparatus and offered neither label nor photo. Checked in `canName` rather than
only at import, **so existing libraries are fixed on sight** with no re-import and no Normalize
run. The same rule already applied to Smith movements.

**`none` is not `bodyweight` (2026-08-15).** `none` is the value a hand-typed exercise is created
with, so it means "unset" far more often than "needs no equipment". Only `bodyweight` suppresses
the label and photo.

**Repeating writes a PLAN, never sets (2026-08-13 onward).** Three routes copy an exercise
forward — "repeat it" on the Last time line, "Do this workout again", "Add to today's workout" —
and all three go through one `copyEntryForToday` / `planFromSets` so they cannot drift. Sets are
cleared and **cardio is nulled**, because `entryDone()` treats any entry carrying cardio as
finished; carrying it over hands back a workout that was complete before you left the house.
Machine setup and per-set notes ride along and stay editable.

**Video frames scale with duration, and that sets the image cap (2026-08-13).** One frame per
2.5s, so 60s of walkthrough is 24 frames — which is *why* `SCAN_MAX_IMAGES` is 24, not the
reverse. Rejected: a fixed frame count, which gave a three-second clip sixteen near-identical
stills, each one billed as a vision image.

**One button wherever the OS sheet already offers the camera (2026-08-13 / 08-25).** The picker
sheet's second item *is* "Take Photo or Video", so a `capture`-forcing button beside it could
only ever do less. Consolidated on scan, gym photos and progress photos. The progress pair was
kept longest because `capture="user"` opens the **front** camera, which the sheet will not
select; traded knowingly — the flip is one tap, against gaining the library in the same button.
**The walkthrough recorder keeps its `capture` and is not a candidate**: it is one tap straight
to filming, not a subset of anything.

**The coach lives where the question is asked (2026-08-28).** `session_design` could always
answer "I'm at River Crossing, ideas for biceps" — it reads the gym's kit, the week's training and
the time available. It went unused for weeks because it sat behind the centre + labelled "Plan
today's workout", which reads as designing a whole session. Rejected: a second, lighter "ideas"
feature, which would have duplicated a working one. Chosen: a door from the exercise picker, the
screen you are on when you do not know what to add. **A plan made during a workout APPENDS to it**
— it used to offer "Start this workout", which was refused because one workout runs at a time, so
the plan was silently discarded.

**The set buttons ACT, they do not tag (2026-09-04). Read this before changing them again.**
This control has now had three designs, and the history is the point. v0.26.0: direction arrows
that MARKED a set. Then Chris asked for feeling — Easy / Struggle faces — on the grounds that how
a set felt is what you can answer honestly a minute later. Now: two buttons that move the weight
by one step there and then, `+5 lb` / `-5 lb`, because he wanted the control to *do* the thing
rather than record an intention. That is a third design, not a return to the first. **Not tapping
means "same again"** — the steppers already carry forward — which is why there is no third button.
The direction still writes to `effort` via the existing `up`/`fail` aliases, because the logged-row
chips, the progression nudge and the 28-day Progress lens all read that field and the real history
holds both spellings; a new field would orphan all of it.

**Sonnet is the floor (portfolio rule).** No task routes to Haiku. Models change from Railway
variables alone — `AI_MODEL_DEFAULT` or `AI_MODEL_<TASK>` — and an unknown value falls back to
the built-in choice and says so in the logs, so a typo never takes the app down.

**Never return an upstream body to the client (2026-08-22).** It can carry the API key. Statuses
and our own sentences only. Logging upstream bodies **server-side is correct and expected** — the
two were conflated once and it cost a diagnosis.

---

## Traps — each one happened

**A revoked API key looks perfectly healthy.** `/health` reported `"ai": "configured"` through a
total AI outage, because that field only ever meant *the env var was non-empty at boot*. Every AI
feature failed and the cause took a round of guessing. `/health` now also carries `ai_last_call`
(`{ok, status, at}`) recording the last **real** relay call. **Check that field first** when AI
misbehaves. Fixed 2026-08-22, backend v0.29.0.

**`AI_PRICES` declared `claude-sonnet-5` twice (2026-08-01, backend v0.10.1).** Python keeps the
last value, which was Haiku's $1/$5 — so Sonnet 5 was metered at **a third of its true cost**,
and the $25 monthly breaker would have passed roughly **three times** the money it believed
before tripping. The same duplication collapsed the table to a single key, and because
`AI_PRICES` doubles as the **model allow-list**, the other four models the picker advertised
would have 400'd every call. This is why `portfolio-audit` check **B2** blocks a push on
duplicate `AI_PRICES` keys. Sonnet is held at the standard $3/$15, *not* the intro $2/$10 that
ran to 2026-08-31, so metering never under-counts once intro lapses — over-counting is the safe
direction.

**The updater lied for eight releases.** Update detection compares `BUILD`, which went unbumped
while `APP_VERSION` climbed 0.74 → 0.81, so "Check for updates" truthfully said "you're current"
while Chris was stranded on an old build. `check.js` now **fails** when `sw.js VERSION` differs
from `APP_VERSION`, and warns on a stale `BUILD`.

**The offline shell could not boot.** `sw.js` primed React and ReactDOM only — but the whole app
is one inline `text/babel` block, so with no `babel-standalone` nothing transpiles and you get a
**black screen carrying no error at all**. It hid because the runtime cache fills these on first
use, but `ASSET_CACHE` is keyed to `VERSION` and `activate` deletes the old one, so **every
deploy reopened the gap**. `check.js` now fails the build if any `<script src>` in `index.html`
is missing from `CRITICAL_ASSETS` or its integrity string has drifted.

**Railway custom domains need TWO records** — the CNAME *and* a `_railway-verify.<sub>` TXT.
Relaying only the CNAME burned 30 minutes proving correct DNS.

**Link-preview crawlers do not run JS.** OG tags for `/g/{token}` are server-rendered, and
`og:image` must be a **stable proxy URL**, not a signed one — a signed link dies in an hour and
survives revocation.

**A note can crush its own set row.** Flex children set to `0 0 auto` beside a flexible sibling
take their width first, and "16 x 55 lb" broke onto four lines. `.set-row.logged` scopes the fix;
plain `.set-row` carries prose rows elsewhere that must keep wrapping.

**Uncontrolled inputs do not refresh.** `defaultValue` fields (machine setup, per-gym labels)
need a `key` that includes the value, or a programmatic change leaves the box stale while the
text above it updates.

**Effects keyed only on `sets.length` miss a plan arriving.** "Repeat it" gives a card a plan
without logging anything, so the count never moves; `planned` belongs in the deps.

---

## What is open

**Blocked on Chris:**
- **Pro billing and quotas** — the only large feature left. Zero Stripe in the backend, no Pro
  gating in the app. Needs his pricing decisions before anything can be built.
- **Log a first bodyweight** — 30 seconds, and it unlocks three built-but-dormant things: the
  check-in body row, both Progress body lenses, and the deficit-aware coaching in the Program
  Builder, which asks age and *derives* the deficit from weigh-ins.
- Import Golf Power 30 rev 5; community-gym round trip (share → find); delete-account on a
  throwaway; Gmail "Send mail as" over Resend SMTP.

**Known gap, not scheduled:** "Normalize my library" renames to canonical titles but does **not**
merge duplicates — which is how one seated cable row became two records with split history.
Merge exists manually (Library → exercise → Merge; deliberately absent from the help panel,
which has no undo channel for a bulk rewrite). Automating it inside Normalize needs a review step
first.

**Parked:** nothing. The roadmap is otherwise clear.

---

## Where authority lives

| Question | Truth |
|---|---|
| What the app does now | `index.html` — there is no other copy |
| Deployed versions | `/health` for the backend; `APP_VERSION` in the served `index.html` |
| Whether a release is safe | `node check.js`, then `portfolio-audit` on push |
| Which model a task uses | `GET /api/ai/models` — reflects Railway overrides, not the source default |
| Why a decision was made | This file, then the comment beside the code |
| Cross-app plumbing | `Forever Apps/PORTABLE-IMPROVEMENTS.md` |

Secrets live in Railway and Dashlane. They are in neither repo, and must never enter a chat, a
commit, or `BRIEFING.md`.
