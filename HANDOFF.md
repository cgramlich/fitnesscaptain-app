# FitnessCaptain — HANDOFF

**State as of 2026-10-03.** App `v0.145.0`, backend `v0.30.0`, both live. Pro billing deployed but SWITCHED OFF (`BILLING_ENABLED` unset); `billing.sql` has been run. Launch steps remaining: see What is open.
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
| App | `v0.140.0` | https://fitnesscaptain.com — GitHub Pages, repo `cgramlich/fitnesscaptain-app` |
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

**In a live workout, every exercise you have not started is collapsed (2026-09-27).** An
untouched card is ~320px of steppers, plan line and note box, so three exercises pushed the third
off the phone and you could not see the shape of the session you were about to do. A collapsed
card shows its plan, its target or last time's sets under the name, so the row still says
something. Anything with a set logged in it stays open - that is the one you are in the middle of.

It shipped first with the exercise you were UP TO left open, on the reasoning that a screen of
shut cards reads as empty and costs a tap before your first set. Chris saw it and said "whole
list": you open a workout to see what you are in for, not to be handed the first movement, and
the tap spent choosing is one you wanted to spend. **Do not reinstate the open-the-first-one
exception** - it was built, seen and rejected by the person using it. Also rejected: a remembered
per-workout preference, which makes the same workout look different on two days for reasons you
cannot see. Reviewing a **past** workout still expands everything - `collapsed` is gated on
`live`, deliberately.

**Logged-set row: reps arrows left of the next chip, weight arrows right (2026-10-03, v0.145.0).**
Chris picked this over a second line per set. The four arrows and the chip are ONE nowrap group:
at 393px+ (current iPhones) it sits on the set's line; at 375px it drops under the set as a
whole rather than splitting a pair from its number. Same release:

- **2.5 lb steps when the history says so** (`inferWeightStep`): any logged, planned or next-time
  weight for that exercise that is a multiple of 2.5 but not of 5 (82.5) makes the jump 2.5 for
  the steppers and the arrows. Evidence only, never an override: an explicit Weight step on the
  exercise wins, and no evidence leaves the 5 lb / 2.5 kg default.
- **Add exercise mid-workout plans from your latest session** (Chris: "add an exercise I just
  thought of from a prior workout and give me the most recent of that exercise"), the same
  `latestEntryFor` + `latestSetupFor` as the coach and Add to today. It used to arrive blank with
  a "repeat it" link.

**Next time is set on the set you just did, not in the logging panel (2026-10-03, v0.143.0).**
Chris: "after a set that I feel is easy, I wanna go up for that same set the next time, but it
looks like the code is just having me change it for the next set." It was: the Next time control
lived in the panel, and the panel belongs to the set you are ABOUT to lift, while "that was easy"
is only knowable after. The control is now on each logged row - a green up arrow (one weight step
per tap) beside a red down arrow (Chris: "so I will never have to open the number"), and the next
chip, which opens one line each for reps and weight as red-down / typeable number / green-up
(`NextNum`, Stepper's typing rules), plus "same as today". The panel line is
GONE (Chris chose one place over two). Rows became `div role=button`: a button inside a button is
invalid and its clicks fall through to the row. `setNextFor` stores numbers, never a direction, and
equal-to-today clears the target. Editing a set in the panel still writes its existing target back.

**The coach is on the front screen, and plans with YOUR numbers (2026-10-03).** Chris wanted a
standout AI feature on the first screen: "I'll be at River Crossing today, look at my recent
workouts, identify body parts I need to hit, build me a 40 minute resistance workout", then tick
the exercises he wants into today's workout. The coach already existed ("Today's session",
`session_design`) but sat behind the centre + and a picker link. Now:

- **Coach card** at the top of Workouts (under an in-progress workout). It is only a door: the
  sentence opens the session builder and is sent on arrival. No mic button of our own - keyboard
  dictation already works in any text box, and a second mic is another permission prompt.
- **Gym from the sentence, by the model.** Every gym is listed in the first turn with its Google
  place name, because people type a nickname ("RCC Gym") and say the real name ("River
  Crossing"); a string match cannot bridge that, the model can. The plan block names the gym and
  the picker follows it. The last-used gym is stated as a DEFAULT, never as "Training at" - as a
  fact it would outrank the gym named in the request.
- **Time from the sentence, on the device** (`parseAskMinutes`): the budget is given to the model
  as a fact, so a picker saying 45 beside a sentence saying 40 would plan to the wrong number.
- **Days since each muscle group**, the same `buildDigest().daysSince` the Progress check-in
  shows, so the coach and the card cannot disagree. It opens with the overdue groups by number.
- **Tickable plan**, everything ticked by default (Chris's call: you keep most of a plan you asked
  for). The button carries the count: "Add 3 exercises to today's workout".
- **Your numbers, not the coach's.** Any exercise with history arrives planned from your latest
  session - sets, next-time targets, setup at that gym - via the same `latestEntryFor` /
  `latestSetupFor` as Add to today. The card says "Your last: ..." for those rows so it never
  shows 3 x 10 and logs something else. Cost: a plan built to 40 minutes can run long if your own
  scheme has more sets than the coach assumed.
- **First live use found three gaps (2026-10-03, v0.142.0).** Chris: "I am at River Crossing
  today. What am I due for?" The coach asked which gym he meant (his RCC gym has no Google
  link, so nothing tied it to River Crossing) and he could not answer, because the gym and time
  pickers were DISABLED once the chat began. Fixed: (1) the pickers never lock; changing either
  mid-conversation sends a re-plan turn, shown as "Changed to 30 min". (2) Questions come with
  tappable answers - an ```ask block parsed like ```plan - and the coach must ask when the time
  was not stated (said in the sentence or set on the picker by hand) or the gym is genuinely
  unclear. (3) **Gym aliases are learnt**: the plan block's `heard_as` is saved on the gym
  (`gym.aliases`, last 5) and listed as "also called" in every later session, so a name costs
  one question, once.

**Pro billing: separate lifetime counts, shared Stripe, off until tested (2026-09-28).** Price is
MenuCaptain's: $2.99/month, $19.99/year. Free is a LIFETIME allowance per feature - 40 AI
requests, 3 gym scans, 10 gym searches - each with its own counter, because a scan costs about
ten chat calls and must not be able to eat the chat budget. Chris chose this over one pooled
credit ("why did that cost 8?") and over MenuCaptain's single 75-call counter. Pro removes all
three; the monthly abuse ceilings still apply behind it. Counters start at zero on launch day:
nothing is counted while `BILLING_ENABLED` is off.

How it is enforced, in the backend: the free check runs BEFORE any provider spend and returns
402 with a structured detail (`code: free_limit`, `kind`, `message`); the app turns that into the
Pro sheet. **A scan is whatever carries images**, never the client's task label, so a scan cannot
be sent as a chat to dodge the smaller counter. A use is counted only AFTER the call succeeds - a
provider failure never costs somebody one of their three scans. Cached gym searches are free.
Both guards were proven by breaking them on purpose: the tests fail without them.

**WARNING - shared Stripe account.** MenuCaptain sells from the same MilSpo Life account, and a
Stripe webhook receives every subscription event on the account. Everything FitnessCaptain
creates is tagged `metadata.app = fitnesscaptain`; the webhook ignores anything neither tagged
nor on one of our prices. Without that, a MenuCaptain subscriber would be written into this
app's table. MenuCaptain's own webhook has the mirror-image gap; that finding was routed to the
MenuCaptain session (single-writer), not fixed from here.

Also closed in `billing.sql`: `record_ai_usage` was executable with the public anon key, so
anyone could inflate `system_meter` past the $25 breaker and switch AI off for every user.

**Borrowing an exercise copies your LATEST session of it, not the one you tapped (2026-09-28).**
Chris asked what history came along when he borrowed from an old workout; the answer was "that
day's sets", so a three-week-old curl arrived planned at three-week-old weights with the newer
numbers in grey on the Last time line beneath it. The `+` means "this exercise, today", not "redo
that date". `latestEntryFor()` picks the most recent FINISHED entry with a logged set, sorted by
`startedAt` - the same key as the Last time line, so the two can never disagree. The tapped entry
is only the fallback. The machine setup is looked up SEPARATELY by `latestSetupFor()` - the most
recent finished session that has one, because a setup is written once and rarely repeated, so the
latest session usually lacks it (shipped first without this; Chris: "most recent with a setup").
Scoped to today's gym: a workout at a known DIFFERENT gym is skipped, since seat 3 on one maker's
machine is not seat 3 on another's; a workout with no gym recorded still counts, or every setup
written before gyms existed would be lost. Chosen over labelling the old plan with its date (option B), which
Chris declined. Reviving a whole day exactly as it was is still "Do this workout again".

**Exercise cards swipe left to delete (2026-09-28).** Every other list row in the app already did
- workouts, gyms, routines, and the sets inside the card - so the exercise card was the one place
you had to hunt a small x. It is also the only delete a COLLAPSED card can offer, and collapsed is
now the resting state of anything unstarted. Undo was already there: `removeEntry` keeps the whole
previous array and the toast puts it back. The x stays on the open card. **Nested swipe rows**: a
card swipes and so does each set inside it, so `SwipeRow` now ignores a pointerdown whose nearest
`.swipe-body` is not its own - otherwise dragging a set slid the card too and you could not tell
which delete you were about to hit.

**Search results lead with what you already train (2026-09-28).** This REVERSES an earlier call
that typing a name means you have said what you want, so re-ranking the matches would only move
it. That holds for a full name and fails for the three letters people actually type - "cur"
matches a dozen curls, two of which are yours. Order: what this gym has, then what you have done,
then how well the name matches (a name that starts with the query beats one that merely contains
it - the old worry was real, it just belongs one level down), then sessions and recency. The row
shows its session count while searching even at 1x, because an order you cannot see is a
mysterious one. The separate "Usual" block stays a browsing-only thing.

**Add to today's workout does NOT navigate (2026-09-28).** It jumped to today's workout on the
first tap, so borrowing three exercises off last Tuesday's session was three round trips through
history. Chris: "I want to stay on that page and pick multiple exercises, and then I can go where
I need to." Shopping and checkout are different acts. The toast now carries the feedback the jump
used to give - it names the exercise AND says how many are in today's workout - because you are
adding to something you cannot see, and "added" alone cannot tell you whether the second tap
registered. Same for the branch that starts a workout from scratch: it starts it and stays put.

**Move down exists because bubble-up alone is not discoverable (2026-09-27).** Up-only is
sufficient to reach any order, which is why it shipped alone, but only if you work out that
demoting the second exercise means promoting the third. Both arrows now sit on the open card AND
on the collapsed one - with unstarted cards collapsed by default, arrows only on the open card
would mean opening a card to move it and closing it again. Still not drag-and-drop: a drag handle
fights the scroll on a phone.

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

**The set control targets NEXT WORKOUT (2026-09-05). Read this before changing it again.**
This control has now had three designs, and the history is the point. v0.26.0: direction arrows
that MARKED a set. Then Chris asked for feeling — Easy / Struggle faces — on the grounds that how
a set felt is what you can answer honestly a minute later. Now: two buttons that move the weight
by one step there and then, `+5 lb` / `-5 lb`, because he wanted the control to *do* the thing
rather than record an intention. That is a third design, not a return to the first. **Not tapping
means "same again"** — the steppers already carry forward — which is why there is no third button.
The direction still writes to `effort` via the existing `up`/`fail` aliases, because the logged-row
chips, the progression nudge and the 28-day Progress lens all read that field and the real history
holds both spellings; a new field would orphan all of it. Then a fourth: those buttons moved
TODAY'S weight, and Chris wanted next workout's. They also had a real defect that shows why the
current design stores numbers rather than a direction — toggling up then down returned the weight
to its start but left the mark from whichever button was pressed last, so a set logged "went down"
having changed nothing. **What is stored now is `set.next = {reps, weight}`**, absolute, per set,
and only when it differs from what was logged — an untouched set keeps following what you lift
rather than being frozen. `planFromSets` reads it, which is why all three repeat routes pick it up
without knowing it exists. **`effort` is now read-only**: history renders it, imports write it, the
Progress lens counts it, but nothing new sets it.

**Removing `effort` as an input left two orphans (2026-09-05).** Worth knowing as a pattern: when
a signal stops being written, everything that DISPLAYED it silently changes meaning. The thumbs-up
badge meant "finished, nothing to say" — a real distinction while Easy/Struggle existed, noise once
every set qualified. The progression nudge below is the other. Both removed. The direction chips
were kept, because old records still carry marks and there they still mean something.

**The progression nudge is GONE (2026-09-05), and should not come back as it was.** It compared
against the heaviest set of the whole PREVIOUS SESSION rather than the set you were on, so on set 4
of a pyramid at 30 lb it offered 55 — last session's top set of 50 plus a step. Structurally wrong
for anyone who pyramids. It also read `effort`, which nothing writes any more, and it argued with
"Next time", which states the target for that set position explicitly and cannot be wrong about
which set it means. **If it returns: compare set N to set N, and defer to an explicit next-time
target.**

**Weight step is per exercise (2026-09-05).** Same shape as rest — a global default with a
per-movement override — because dumbbells move in 5s, a barbell with micro-plates in 2.5, and a
stack can jump 10. It drives the weight stepper, the next-time targets and the progression nudge.

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

**A destructive action needs an undo CHANNEL, not just an undo function.** Every delete keeps the
whole previous array and offers it back for 12 seconds — but the channel is a prop, and the
ExerciseSheet mounted inside the (i) panel was not given one, so deleting an exercise there
dropped it silently and closed the panel. Delete and Merge are both gated on `undoable` now: a
mount that cannot restore does not show the button. **Check the channel reaches any new mount of
a sheet that can destroy something.**

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
- **Pro billing — BUILT, NOT LIVE (2026-09-28).** Code complete in both repos behind
  `BILLING_ENABLED` (off). Launch sequence, one step at a time, in this order: (1) Chris runs
  `fitnesscaptain-backend/billing.sql` + his comp row; (2) push backend, then app; (3) Chris
  creates the FitnessCaptain product + two prices in MilSpo Life's Stripe **test** mode, a
  webhook to `/api/stripe/webhook` (events: checkout.session.completed,
  customer.subscription.updated, customer.subscription.deleted), keys into Railway only;
  (4) `BILLING_ENABLED=1`, end-to-end test purchase with a Stripe test card; (5) swap to live
  keys. `/health` → `billing` shows enabled / ready / test-or-live at each step.
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
