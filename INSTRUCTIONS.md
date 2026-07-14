# Workout Tracker — How the Loop Works

This is the canonical how-to for running Austin's workout tracker. Any Claude
session (desktop or phone/cloud) that Austin texts about working out should read
this first. Read `decision_log.md` before changing code/data.

---

## The one-URL app (GitHub Pages)

**Permanent app URL:** https://austineckk.github.io/workout/
**Source of truth for the app:** `index.html` at the root of the
https://github.com/austineckk/workout repo (public, Pages serves main).
Everything the loop needs lives IN this repo — library, demos, history, these
instructions — so any session with the repo can run the full loop: desktop
sessions via the local clone at `Workout Tracker/site/` on Austin's PC, and
cloud/phone sessions via GitHub directly. On desktop, ALWAYS `git pull`
before reading or writing — phone sessions push too.

> **History note (2026-07-14):** the app originally lived at the claude.ai
> artifact https://claude.ai/code/artifact/51d9ead8-a359-4b89-aaea-9d9ee3ac3d68,
> but that URL sits behind a claude.ai login wall (Austin's Safari is signed
> into his PERSONAL account, the artifact belongs to the WORK account, and his
> workout partner has neither). Moved to GitHub Pages — public, no login,
> read-only to the world. Do NOT push new workouts to the old artifact.

The app only ever holds the CURRENT planned workout. It is single-file,
self-contained, zero external requests, iPhone-first, dark+light.
Austin and his workout partner both open the SAME URL on their own phones
(partner's data is not tracked yet). History lives in files, never in the app.

**To push a new/updated workout:** `git pull`, edit `index.html` (swap the
`#workout-data` and `#exercise-demos` JSON blocks), then
`git add -A && git commit -m "<focus> YYYY-MM-DD" && git push` — **commit
directly to `main` and push; never open a PR or feature branch** (Pages only
serves main, and there is no reviewer). Pages redeploys automatically in
~30–60 s; Austin just refreshes the app. The old claude.ai Artifact-tool flow
is retired.

---

## The daily loop

```
1. PROPOSE   Austin texts the vibe ("legs today, ~40 min, shoulder's a bit cranky").
             Claude proposes ONE best-option plan with pivot alternatives
             ("I'm thinking X; if you'd rather hit Y, we can shift"). No 20 questions.

2. PUSH      On approval, Claude builds the workout JSON (see schema below),
             hydrates coaching fields + demos from the library, and publishes to
             the SAME artifact URL. Old workout disappears from the app on push.

3. TRAIN     Austin works out, tapping sets done and editing reps/weight as he goes.
             Anything is skippable; timers are optional and never block progress.

4. SUMMARIZE Post-workout Austin taps "Copy my workout summary" (bottom of the app,
             and on the session-complete card) and pastes the block back into chat.

5. STORE     Claude parses the pasted summary and writes ONE history file:
             history/YYYY-MM-DD-<focus-slug>.json  (schema: history/_schema.json).

6. PROGRAM   Next session, Claude reads the history files, applies the guardrails,
             and drafts the next workout ("look at my past workouts, tell me what to
             hit next") — then back to step 1.
```

---

## Building a workout to push (workout-data schema)

Embedded in the artifact as `<script type="application/json" id="workout-data">`.
Top level: `{ id, date, focus, format, blocks[] }`.

- `id` — unique, convention `YYYY-MM-DD-<focus-slug>` (matches the history filename stem).
- `date` — ISO `YYYY-MM-DD`. Always present.
- `focus` — human label, e.g. "Upper Push/Pull + Core".
- `format` — primary format: `straight` | `circuit` | `emom` | `amrap` | `for-time`.
- `blocks[]` — each: `{ id, title, kind, rounds, note, exercises[] }`
  - `kind` ∈ `straight | circuit | emom | amrap | for-time`.
  - optional format fields: `interval_seconds` (EMOM, default 60),
    `cap_seconds` (AMRAP default rounds×60 / for-time default 0 = open stopwatch).
  - `exercises[]` — each carries the fields the app needs offline:
    `id, name, muscle_group, reps, weight_lb, is_timed, rest_seconds,
     shoulder_pain_flag, per_side, setup, cue, mistake, youtube`.
    (`reps` = seconds when `is_timed`. `weight_lb: null` = bodyweight.)
  - Set-rows shown per exercise = `block.rounds`.

**Hydration (don't hand-write coaching text):** the programmer only decides
`id / reps / weight_lb / rounds / kind / notes`. The coaching fields
(`setup / cue / mistake / youtube / shoulder_pain_flag / is_timed / per_side`)
come from `exercise_library.json` keyed by `id`; splice them in at push time.

**Demos:** after writing `#workout-data`, regenerate the scoped
`<script id="exercise-demos">` map to just this workout's unique ids, pulling each
SVG from `exercise_demos.json` (keeps the push lean — see decision_log Sprint 4).
Every library id has a demo.

---

## The summary the app produces (parse convention)

The "Copy my workout summary" button generates a plain-text block. Format
(v1 — the last line carries the machine tag):

```
🏋️ WORKOUT SUMMARY
Paste this whole block back into any Claude chat to log the workout.
(✓ done · ○ not done · weights in lb, @ = load, s = seconds held)

Date: 2026-07-14 (Tue)
Focus: Full-Body Conditioning Base
Format: Circuit
Duration: ~34 min (planned estimate — tell me the real time if you tracked it)
Feel: (optional — energy, appetite, right shoulder, mood)

A · LOWER STRENGTH — Straight sets · 3 rounds
  Barbell Back Squat [back_squat] (legs): ✓8@135lb  ✓8@135lb  ✓8@140lb
  One-Arm Dumbbell Row [db_row] (back, each side): ✓10@45lb  ✓10@45lb  ✓10@45lb
...
C · CORE CIRCUIT — Circuit · 2 rounds  [SKIPPED]
...
  Child's Pose [childs_pose] (stretch): ○ skipped

SKIPPED: C · Core Circuit (whole block), Child's Pose

── wt-summary v1 · id=2026-07-14-full-body · focus-slug=full-body-conditioning-base · done 22/22 sets ──
```

**How to parse it into a history record:**
- Each exercise line: `<name> [<id>] (<muscle_group>[, each side]): <tokens>`.
- Each set token: leading `✓` = completed, `○` = not completed. Body is the
  actual value AFTER Austin's edits: a number = reps; `Ns` = an N-second timed
  hold; `@Xlb` appended = working weight X lb; no `@` = bodyweight (`weight_lb: null`).
  So `✓8@140lb` = 8 reps at 140 lb, done. `○30s` = 30-second hold, not done.
- `[SKIPPED]` on a block header or `○ skipped` on an exercise → `skipped: true`,
  empty `sets`. Also listed in the `SKIPPED:` line and the record's `skips[]`.
- `Duration:` is the app's PLANNED estimate, not measured. Store `duration_min: null`
  unless Austin gives a real number; note the estimate in `feel_note` if useful.
- `Feel:` — whatever Austin typed/said. Optional → `feel_note`.
- Machine tag gives `id`, `focus-slug`, and a `done N/total` checksum to validate the parse.

Missing/garbled block? Ask Austin rather than guessing. The summary is the record
of intent; Austin's chat message can override any field ("actually it was 52 min").

---

## Storage conventions

All paths are relative to the repo root (`Workout Tracker/site/` on Austin's PC).

- **History:** `history/YYYY-MM-DD-<focus-slug>.json`, one file per workout,
  conforming to `history/_schema.json`. `date` always present. This is the
  source of truth for programming. `<focus-slug>` = focus lower-cased, non-alphanumerics
  → hyphens (matches the summary's `focus-slug=` tag). Committed and pushed like code —
  that's how desktop and phone sessions share it.
- **`_`-prefixed files are NOT real workouts** — they're schema/test/demo files
  (`_schema.json`, `_looptest-*.json`). The programmer MUST ignore underscore-prefixed
  files when reading past workouts.
- **Library:** `exercise_library.json` (61 entries) — the exercise bank. Grows organically.
- **Demos:** `exercise_demos.json` (SVG form drawings, one per id). The generator
  (`tools/gen_demos.js`) and the build-era files stay desktop-only in the Dropbox
  `Workout Tracker/` folder, one level up from the repo.
- **Log:** `decision_log.md` — every load-bearing decision. Desktop-only (Dropbox
  `Workout Tracker/`, one level up). Read it before changing code when on desktop;
  cloud sessions changing app code should note decisions in commit messages instead.

---

## Guardrails digest (full version in the backlog's frozen plan)

- **Goal:** lifelong healthy routine + winning appetite back. Former D1 decathlete,
  6'3" ~160 rebuilding toward 175+. **Conditioning base first.**
- **Split:** rotate chest / shoulders / legs / back; **core most days**; look at recent
  history and hit what was under-served or skipped last time.
- **Stretching is first-class** — in every warmup and cooldown; on-demand desk-stretch
  "workouts" can be pushed to the same artifact.
- **Right shoulder-aware.** Pain map (0–4): overhead press 2, lateral raise 3, bench 1.5,
  incline 2, pull-ups 3.5, dips 2. Program AROUND and INTO it — 3–4 min shoulder prehab
  in most warmups, prefer incline over flat, favor face-pull / rear-delt / band work over
  the painful lateral raise. Strengthen, don't aggravate.
- **Barbell lifts fully in** (deadlift, front/back squat, barbell lunge, step-ups, OHP,
  LIGHT hang cleans as warmup/conditioning only) — but **NEVER heavy cleans/snatches**
  (broken-radius history). Hip-explosive work via DB/KB swings, jump shrugs, med-ball throws.
- **Equipment (Irondale, confirmed):** DBs to 100 lb, squat rack + barbell/plates, flat+incline
  bench, Smith, Matrix strength + cable machines, TRX, full cardio line. NOT confirmed
  (substitute until Austin verifies in person): kettlebells, plyo box, rower, med ball, jump rope.
- **Timing:** warmup 5–10 min, workout 35–45 min typical. Each block shows a time estimate;
  handle "shorter today" by skipping blocks, not by re-planning mid-gym. Real re-programming
  happens in chat.
- **Units:** pounds. **No gamification, no streaks.**

---

## Cloud / phone sessions

Phone sessions run in a cloud environment hooked to Austin's GitHub with this
repo — which contains everything the loop needs. Rules:
- **Full loop works anywhere the repo is available.** Read library/history from
  the repo, edit `index.html`, write history files, commit to `main`, push.
  No PRs, no feature branches — Pages serves main directly.
- **Pull before you touch anything** (`git pull`) — desktop and phone both push.
- **If a session somehow can't push** (missing credentials): still PROPOSE the
  workout and hand Austin the finished plan so pasting it into a desktop session
  is a one-liner; and for STORE, his pasted summary is self-contained — the next
  session with repo access writes the history file. Nothing is lost.
- The app never needs a backend to run — it's fully self-contained on device
  and persists check state in localStorage keyed by workout id.
