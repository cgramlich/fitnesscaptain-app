# FitnessCaptain — BRIEFING

**Deck-ready. Written to leave the machine.** Derived from `HANDOFF.md`, which is authoritative
if a figure here disagrees. Current as of **2026-08-28**, app v0.122.0 / backend v0.29.0.

> **Every training example in this file is fabricated** to match the real shape of the data.
> No actual workout history, bodyweight, or health metric appears here, by design.

---

## The arc

**Situation.** Someone who lifts in more than one place — a home rack, a club, a hotel gym on the
road — accumulates a training history that looks continuous and isn't. Every general-purpose
logger records "Lat Pulldown, 100 lb" as one number on one line.

**Problem.** That number is not one number. A weight stack marked 100 on a Life Fitness machine
and one marked 100 on a Hammer Strength differ by pulley ratio, cam profile and plate
calibration. So a chart that plots them together shows progress that never happened, and hides
progress that did. The obvious fix — a database of gym equipment mapping machine to true load —
does not exist in any usable open form, and manufacturer specifications are proprietary and rot.

**What was done.** The problem was re-framed from global to **local**. The app doesn't need to
know what every machine in the world does; it needs to know which room you are standing in. Every
session, every personal record and every machine label carries its gym. When a trend spans two
gyms on machine-loaded work, the app says so rather than inventing a conversion factor. Free
weights are exempt, because 45 lb of plate weighs 45 lb everywhere — the warning fires only where
it is true.

**What it means now.** A working, deployed, daily-use application: workout logging, a
873-exercise reference library with illustrations and demo video, AI-assisted session planning, a
camera-based gym equipment scanner, gym sharing by link, and offline operation. Built and shipped
by one person working with an AI pair, at roughly one release per working day across August 2026.

---

## Figures — all with units and dates

| Figure | Value | As of |
|---|---|---|
| Reference exercise library | **873 movements**, with equipment, muscle group and written how-to | shipped |
| Application size | **~10,460 lines in a single HTML file**, no build step, no bundler | 2026-08-28 |
| Gym scan input | **24 images per scan** = 60 seconds of walkthrough at one frame per 2.5s | 2026-08-13 |
| Gym scan cost | **~$0.08 per scan** (~26,000 input tokens, Sonnet-class vision) | 2026-08-13 |
| AI spend ceiling | **$25/month** hard breaker, plus a per-user call cap, enforced before spend | live |
| Metering defect found and fixed | a duplicate price key metered one model at **1/3 of its true rate** | 2026-08-01 |
| Offline boot dependencies | **4 of 4** third-party scripts pre-cached with integrity hashes | 2026-08-15 |
| Release cadence | app v0.13 → v0.122 over ~6 weeks | Jul–Aug 2026 |

---

## Quotable claims — each survives a follow-up question

> "A weight stack marked 100 is only 100 on that machine. The app knows which room you're in, so
> it never pretends two gyms are the same gym."

> "We solved cross-gym comparability without an equipment database, because no usable one exists.
> The problem was local, not global."

> "Film a walk around the room and the app writes the equipment list."

> "The AI has a hard monthly spend ceiling enforced before the call is made, not after the bill
> arrives."

> "A duplicate line in a pricing table meant one model was metered at a third of its true cost.
> The budget guard would have passed three times the money it thought it had. It is now a
> blocking check that no release can bypass."

> "It gives no calorie, macro or medication advice, and never suggests a rate of weight loss.
> That restraint is written into the model instructions, not left to chance."

> "Open it with no signal and it still works. Your history is on the device."

---

## What deserves a picture

**One number, two rooms** *(the strongest single image)*. The same exercise, same recorded
weight, two different machines — with the app's warning between them. This is the entire thesis
in one frame. Fabricated illustration: *Lat Pulldown, 100 lb at Gym A vs 100 lb at Gym B, flagged
as not comparable.*

**Walkthrough to inventory.** A film-strip of sampled video frames on the left, the structured
equipment list they produced on the right. Shows a vision feature doing something concrete.

**The sampling curve.** Clip length against frames extracted, flat-lining at 24. Makes a design
decision legible: a 3-second clip used to yield 16 near-identical stills, each one billed.

**Before/after of a spend guard.** The duplicate-key incident as a two-panel: what the meter
believed vs what was actually being spent. A governance story, not a feature story.

**Architecture, one diagram.** Single-file PWA → metered relay → model, with the key never
leaving the server and the upstream response body never returned to the browser.

---

## Angles, so any audience can be served

**Product.** Solves a real problem the incumbent loggers get wrong, for anyone who trains in more
than one place — which includes most people who travel for work.

**Engineering.** Deliberate constraint: one file, no build step, no bundler, offline-first. Every
non-obvious decision is documented beside the code with the road not taken. Failures become
blocking pre-push checks rather than remembered rules — the metering bug, the version-skew bug
and the offline-boot bug are all now gates that fail a release.

**Governance and cost.** AI spend is metered per user against a monthly ceiling enforced *before*
the provider call. Model choice per task is changeable from a deployment variable with no code
change. Upstream error bodies are never returned to the client, because they can carry the API
key. Health reporting was upgraded after a real incident to report whether the last AI call
actually *worked*, not merely whether a key was configured.

**Process.** Built conversationally with an AI pair, verified by driving the real UI in an
instrumented harness rather than by assertion. Bugs were found by using it — a crushed set row, a
squeezed image budget, a silently rejected key — and each fix carries the incident that motivated
it.

---

## Do not say

- **No real training data, bodyweight, body-composition or health metrics.** These are personal
  health information. Every example must be fabricated and labelled as such.
- **No Pro pricing, tiers or launch date.** Undecided as of 2026-08-28; billing is unbuilt.
- **No API keys, variable values, backend hostnames, database URLs or internal endpoints.**
- **Do not call it a medical, nutrition or coaching product.** It deliberately refuses that
  advice.
- **Do not claim App Store availability.** It is a web app; store distribution is not planned
  for this product.
- **Do not imply multi-user scale.** It is a personal application; the community gym feature is
  built but not yet exercised across accounts.
- **Do not present the AI as autonomous programming.** It plans and organises against your own
  logged data, and you review everything before it saves.
