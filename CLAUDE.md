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
2. Bump `CACHE` in `sw.js` (v21 as of this writing — always +1)
3. Commit with inline identity (do NOT touch global git config):
   `git -c user.email="eric.t.martinez@gmail.com" -c user.name="epunk-ux" commit -m "..."`
4. `git push` (Eric asks for every push; "ship" means push live)
5. Tell Eric the new version number; he pulls-to-refresh on his phone

`gh.exe` lives at `C:\Program Files\GitHub CLI\` (not on PATH) if GitHub API work is ever needed; plain git push has been sufficient.

## Current program (v2 store — the 4-day rotation, reworked 2026-10-21/22)

Six lifts, **each performed twice per round**, same weight both exposures, +5 lb when the round is marked green.
12 exposures fit exactly into 4 days × 3 lifts, so **nothing is hit 3x in a round**:

- **Day A — Squat + Pull:** Zercher squat, Barbell bent row, Pulldown (over)
- **Day B — Hinge + Press:** Romanian deadlift, Overhead press, Pulldown (over)
- **Day C — Squat + Pull:** Zercher squat, Barbell bent row, Pulldown (under)
- **Day D — Hinge + Press:** Romanian deadlift, Overhead press, Pulldown (under)

All 3×6. Exercise IDs: `zercher`, `ohp`, `rdl`, `bb_row`, `lat_over`, `lat_under`.
Barbell lifts (plate hint on): zercher, ohp, rdl, **bb_row**. Pulldowns are machine (no hint).
Display names are kept short on purpose: at 375px phone width the name column is only **142px**, and anything longer
wraps to two lines and makes the row taller. "Pulldown (over)" measures 115px, "Pulldown (under)" 126px, while
"Lat pulldown — overhand" was 188px and wrapped. Measure before renaming any exercise.

**Structure (do not "improve" it — it is constraint-driven, not arbitrary):** every day is 1 lower lift + 2 upper.
The hard rule, set 2026-10-22: **RDL and the barbell bent row never share a day.** Both hammer the erectors and the
grip, and rowing after RDLs means rowing with a shot grip and a fried lower back. So the row lives on the two Zercher
days and the press lives on the two RDL days — "Zercher days pull, RDL days press". The accepted cost is that OHP
follows RDL both times; that is the cheaper conflict, because OHP uses **no grip at all** and the delts are fresh after
a hinge. Also holds: no day carries both pulldown grips, all four days are distinct, and the lower lifts alternate
Zercher → RDL → Zercher → RDL.

Rejected alternatives, so they do not get re-proposed: **reordering Day D to row-then-RDL** only reverses who pays —
you would then RDL fatigued, and RDL is the heavier, more technical lift where fatigue means form breakdown under load.
**Dropping back to 3 days is impossible**: 6 lifts × 2 = 12 slots over 3 days forces one day to carry both Zercher and
RDL, and then the row's two exposures have nowhere to go that is not an RDL day. 4 days is what buys the separation.

**Bench press was removed 2026-10-21 because it was hurting Eric's shoulders, so there is
deliberately NO horizontal push in the program. Do not add one back (bench, dumbbell press,
machine press, push-ups) unless Eric asks.** The barbell bent row was added the same day and
restores the horizontal pull. Pull-to-press volume is now 6:2, which is intentional — heavy
pulling and minimal overhead work is the point while the shoulders settle.

## Data model (localStorage)

- Key: **`regimen.v2`** — `{ cycle: 'Regimen 2026-2', data: { w0: {...}, w1: {...} } }`
- Per week object: `{ <exId>: number, <exId>_inc: true, <exId>_hold: true, bw: number }`
- **One value per exercise per week**, even though a lift appears on two days — the two rows are views of the same value. `exRowRefs` + `syncEx(exId)` in `renderDays()` keep all rows for an ID (input, done state, plate hint, buttons, day counts) in sync live.
- `regimen.v1` is the retired first program (12 exercises incl. curls/laterals/rows, old IDs like `trap_bar_dl`). Orphaned but intentionally left on-device. Never reuse the v1 key.
- **The UI calls them "Rounds"; the code and storage still call them weeks. This split is deliberate — do not "fix" it.**
  A round is one pass through Day A → B → C → D. With 4 days that often takes more than 7 calendar days, so "Week" was
  wrong on screen. Renamed 2026-10-21, **display text only** — four spots: the week-rail pill, the Progression column
  headers (`R0`..`R12`), the bodyweight meta line, and the CSV header. **Storage keys stay `w0`..`w12`**, and the internal
  names (`currentWeek`, `WEEKS`, `weekHasAny`, `renderWeekRail`, `.week-pill`) stay "week". Renaming the storage keys
  would orphan every value Eric has logged.
- Rounds run 0–12 (13 rounds per cycle).
- **Round 0 is the calibration round, not a training round — do not remove it.** Its rail pill reads **"Base"**
  instead of "Round" (a `w === 0` check in `renderWeekRail()`), so it is visibly not one of the 12. Eric spends one pass doing 3 sets of
  each lift to find his starting weights. That is why the cycle is **12 rounds (R1–R12)** even though there are 13 slots:
  Round 0 sits outside it. It is load-bearing, not filler — `todaySuggestion` for Round 1 reads directly from Round 0,
  so deleting it would leave Round 1 with no Today value and break the first link of the +5 chain. Round 0 itself
  correctly shows no Today (nothing precedes it). Marking Round 0 green makes Round 1 suggest the calibrated weight +5;
  marking it orange or leaving it unmarked makes Round 1 suggest the calibrated weight as-is.

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
v19: always-visible plate hint. **v20 (commit `95aca5b`): full rebuild to the 6-lift 3-day rotation, synced shared rows, fresh `regimen.v2` store.**
**v21 (current): Bench press removed entirely (shoulder pain) and barbell bent row (`bb_row`) added. Program went 6 lifts / 4 days / 3 lifts per day, every lift 2x a week. `regimen.v2` key kept, so the five carried-over lifts keep their history; orphaned `bench` values stay in the store, unused, and `bb_row` starts with no history.**
