# Workout Tracker

This repo IS Austin's workout app: `index.html` is a self-contained phone-first
workout checklist, served publicly by GitHub Pages at
**https://austineckk.github.io/workout/**. Austin (and his workout partner) open
that URL at the gym.

**Before doing anything, read `INSTRUCTIONS.md` in this repo** — it is the
canonical how-to for the whole loop (propose → push → summarize → store →
program), the workout-data schema, the summary parse convention, storage rules,
and Austin's programming guardrails (shoulder, no heavy cleans, stretching,
conditioning base).

Quick rules that override any default behavior:

- When Austin gives a workout vibe ("legs today, 40 min"), propose ONE plan with
  a brief pivot option, and on his approval push it. Don't interrogate him.
- Push = edit the `#workout-data` + `#exercise-demos` JSON blocks in
  `index.html`, hydrating coaching fields and demos from
  `exercise_library.json` / `exercise_demos.json` by exercise id.
- **Commit directly to `main` and `git push`. NEVER open a PR or create a
  branch** — Pages serves main and there is no reviewer. `git pull` first.
- When Austin pastes a "🏋️ WORKOUT SUMMARY" block, parse it per
  INSTRUCTIONS.md and write `history/YYYY-MM-DD-<focus-slug>.json`, then
  commit+push that too. History files ARE the training record — never skip.
- Ignore `_`-prefixed files in `history/` when reading past workouts.
- The app must stay a single self-contained `index.html`: no external requests,
  no build step, no frameworks, no new files it depends on at runtime.
