# Regimen — Handoff / Standing Context

Eric's personal workout-tracking PWA. Single-file app, no build step, no backend.

## What this is

- **Live app:** https://epunk-ux.github.io/Regimen/ (GitHub Pages, auto-deploys from `main`, ~30s after push)
- **Repo:** https://github.com/epunk-ux/Regimen.git (public)
- Eric has it installed as a PWA on his phone. Updates arrive via pull-to-refresh — **no reinstall ever needed**, but only if the service-worker cache version is bumped (see Release checklist).
- Data lives in **localStorage on his phone**. Nothing here on desktop is his real data. CSV export (Progression tab) is the backup path.

## Files

| File | Role |
|---|---|
| `index.html` | The entire app — markup, CSS, and all JS inline |
| `sw.js` | Service worker, cache-first. `const CACHE = 'regimen-vNN'` **must be bumped every release** or phones keep the stale version |
| `manifest.webmanifest` | PWA install metadata |
| `icon.svg`, `icon-192.png`, `icon-512.png` | Dumbbell icons (PNGs were drawn with Pillow; Cairo isn't available on this Windows box) |

## Release checklist (every change)

1. Edit `index.html`
2. Bump `CACHE` in `sw.js` (v20 as of this writing — always +1)
3. Commit with inline identity (do NOT touch global git config):
   `git -c user.email="eric.t.martinez@gmail.com" -c user.name="epunk-ux" commit -m "..."`
4. `git push` (Eric asks for every push; "ship" means push live)
5. Tell Eric the new version number; he pulls-to-refresh on his phone

`gh.exe` lives at `C:\Program Files\GitHub CLI\` (not on PATH) if GitHub API work is ever needed; plain git push has been sufficient.

## Current program (v2 — the 3-day rotation, shipped 2026-08-08)

Six lifts, **each performed twice a week**, same weight both exposures, weekly +5 lb progression:

- **Day A — Squat + Pull:** Zercher squat, Bench press, Lat pulldown (overhand), Lat pulldown (underhand)
- **Day B — Press:** Romanian deadlift, Bench press, Overhead press, Lat pulldown (underhand)
- **Day C — Lower:** Zercher squat, Romanian deadlift, Overhead press, Lat pulldown (overhand)

All 3×6. Exercise IDs: `zercher`, `bench`, `ohp`, `rdl`, `lat_over`, `lat_under`. Barbell lifts (plate hint on): zercher, bench, ohp, rdl. Pulldowns are machine (no hint).

Design rationale: minimalist "big five" template (squat/hinge/hpush/vpush/vpull) plus a second pulldown grip. Zercher lands Mon/Fri for recovery; Day C is the heavy day before the weekend.

## Data model (localStorage)

- Key: **`regimen.v2`** — `{ cycle: 'Regimen 2026-2', data: { w0: {...}, w1: {...} } }`
- Per week object: `{ <exId>: number, <exId>_inc: true, <exId>_hold: true, bw: number }`
- **One value per exercise per week**, even though a lift appears on two days — the two rows are views of the same value. `exRowRefs` + `syncEx(exId)` in `renderDays()` keep all rows for an ID (input, done state, plate hint, buttons, day counts) in sync live.
- `regimen.v1` is the retired first program (12 exercises incl. curls/laterals/rows, old IDs like `trap_bar_dl`). Orphaned but intentionally left on-device. Never reuse the v1 key.
- Weeks run 0–12 (13-week cycle).

## Core mechanics — DO NOT break these semantics

- **Today suggestion** (`todaySuggestion`): most recent prior week's value; **+5 only if that week was marked green (inc)**. Shown green with ↑ when bumped, otherwise orange.
- **Orange `=` button (hold):** copies Today into the field. Means "same weight next week."
- **Green `↑` button (inc):** copies Today into the field AND marks intent → **next** week's Today = this value + 5. It does not add 5 to the current field.
- The two buttons are a **mutex toggle** — one, the other, or neither. Untoggling clears the mark without touching the field.
- **Plate hint:** `BAR_LB = 45`, `PLATES_LB = [45, 25, 10, 5, 2.5]` — **his gym has no 35s** (removed deliberately). Hint is always visible on barbell rows: uses the field value, falls back to Today when empty. Unreachable weights show `(≈)`.
- **0 on a barbell exercise = "just the bar"** — shows bar-only hint while typing, snaps to 45 on blur.
- Bodyweight is a separate weekly field (`bw`), shown above Day A and as the first Progression row.
- Progression tab: 13-week grid, green cells where value rose vs. previous logged week; CSV export; Reset-all (confirm-guarded).

## Eric's working style (observed over ~20 releases)

- Small, direct changes shipped immediately — one feature per release, no frills, no refactor detours.
- He describes behavior conversationally and sometimes revises after seeing it live ("actually, change X") — ship fast, iterate.
- Mobile-first: he judges everything on his phone. Buttons were deliberately thinned once (28px wide) — keep controls compact.
- When behavior semantics are ambiguous (what a button writes, what "Today" reads), confirm before coding — the green-button semantics were corrected once mid-flight.

## Version history (abridged)

v1–v13: initial build, plate hints, bodyweight, increase/hold buttons, Today calc.
v14: 0-means-bar. v15: green button also copies Today. v16: "Deadlift" rename.
v17: lateral 1×20, row→machine, dropped 35 lb plates. v18: curls 1×20.
v19: always-visible plate hint. **v20 (current, commit `95aca5b`): full rebuild to the 6-lift 3-day rotation, synced shared rows, fresh `regimen.v2` store.**
