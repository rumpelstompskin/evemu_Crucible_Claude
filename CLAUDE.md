# CLAUDE.md

Personal fork of EVEmu Crucible (an EVE Online server emulator, C++). See `README.md` for setup and
`doc/ChangeLog.md` for what this fork adds on top of upstream.

## How work happens here

- **Build/run:** Docker only (`docker compose up -d --build`). Cloud sessions can't run the game client,
  so changes are usually pushed untested; the owner tests them in game.
- **In-game test checklist:** `ToTest.md`. Add a checklist item for every user-visible fix. The owner
  writes what they actually observed as `//` notes under the item; treat those notes as bug reports.
- **ChangeLog:** user-facing changes go in `doc/ChangeLog.md` under the current version.
- **Memory:** local Claude Code memory on the owner's machine is not visible to cloud sessions. Anything
  the next session needs to know goes in the worklog below.

## Worklog rules

1. **Read the Open threads and the latest Worklog entries before starting.** Check whether your task
   touches something in progress or something already tried and rejected.
2. **Before you finish, update this file in the same commit as your change** (or in a follow-up commit if
   the session had no code change):
   - Add a Worklog entry at the top: date, branch, commits, what changed and **why**, what was verified
     (compiled? tested in game?), and anything tried and abandoned.
   - Record what the owner reported seeing in game, in their words where possible. These observations
     are the record that gets lost between sessions.
   - Update Open threads: add new ones, close resolved ones (move the outcome into the Worklog entry).
3. Keep entries short, a few bullets each. Point to commits for the details; don't paste diffs.
4. If a claim in an older entry turns out to be wrong, correct it in the new entry instead of rewriting
   history.

## Open threads

- **Warp landing position** (branch `claude/wonderful-cori-veosq7`, commit `d4c326a`, not on `master`
  yet). Reverted the distance-based early warp exit from `867342d`. Not compiled, not tested in game.
  Next: owner tests the Warp / Movement items in `ToTest.md`. Expected result: the ship exits warp at the
  target point and then drifts about exit speed × agility past it (a few hundred metres up to ~1 km).
- **"Up to 75 km" warp overshoot** (claimed in `867342d`). Unverified: the coast formula gives ~1 km for a
  battleship, so if a large overshoot is real it has another cause. Needs an in-game reproduction (ship
  type, target, distance) before anyone touches the math again.
- **Station warp-in at 0 km** (`aeee156`). The owner's `ToTest.md` note says ships landed inside the
  station and were flung out to 30 km; the expected landing is 2–2.5 km out. `aeee156` enforces a
  2500 m minimum stop distance. Not confirmed in game yet.
- **0.8.7 checklist:** almost every item in `ToTest.md` is still unchecked.

## Code map (areas touched recently)

- Warp: `src/eve-server/system/DestinyManager.cpp`: `WarpTo()` (target point and stop distance),
  `InitWarp()` (accel/cruise/decel plan), `WarpAccel/Cruise/Decel/Update/Stop()`, and
  `SetSpeedFraction()`/`UpdateVelocity()` (WARP→GOTO handoff and post-warp decel).
- Warp-in points per object type: `BeyonceBound::CmdWarpToStuff()` in `src/eve-server/ship/BeyonceService.cpp`.
- Noticed but deliberately left alone (upstream code): `WarpAccel()` subtracts the cumulative distance
  from `m_targetDistance` every tick, and `WarpUpdate()`'s comment admits ~1000 m of error. `WarpDecel()`
  recomputes the remaining distance from `total_distance`, so the landing point isn't affected.

## Worklog

Newest first.

### 2026-10-30 · `claude/wonderful-cori-veosq7`
- Added this file. Sessions before this date left no notes; the entries below are reconstructed from
  commit messages and `ToTest.md`, so they record what changed, not what was observed in game.

### 2026-10-23 · `claude/wonderful-cori-veosq7` · `d4c326a`
- Owner reported: "When we warp and land at a position, the position we land at is wrong. The math got
  broken."
- Cause: `867342d` added a second warp-exit condition in `WarpDecel()` (remaining distance ≤
  `m_speedToLeaveWarp × m_shipAgility`). The decel curve's remaining distance shrinks ~e× per tick while
  speed is still far above the exit speed, so it fired early. The server dropped the ship out of warp short
  of the target, clamped speed, and sent `CmdGotoDirection` while the client was still mid-warp.
- Fix: back to the speed-only exit (`currentShipSpeed <= m_speedToLeaveWarp`). Not compiled, not tested
  in game.

### 2026-04-18 · `master` · `aeee156`, `867342d` (reconstructed)
- `aeee156`: minimum 2500 m stop distance when warping to stations (ships were landing inside the station
  and getting ejected ~30 km). In `Follow()`, the stop check now uses the raw double distance, fixing
  approach oscillation when follow distance is 0.
- `867342d`: early warp exit to "leave GOTO stopping-distance runway". **Reverted in `d4c326a`**: it made
  ships land short.
- Market seeding: fixed `IDS_PER_REGION` (2M→8M), parallel-seeding retry and lock timeouts, and order
  `issued` timestamps (orders were invisible in game).

### 2026-04-10 → 2026-04-16 · `master` (reconstructed; see `doc/ChangeLog.md` 0.8.7)
- Warp: removed the 10 km overshoot in `WarpStop()`, send `CmdGotoDirection` on WARP→GOTO (fixed the
  180° flip), reset movement state in `InitWarp()` (re-warp desync), removed `Halt()` after warp.
- Docking oscillation (uint32 underflow in `Follow()`), logout position not saved.
- Drones, wormhole scanning/mass/collapse, anomaly visibility/respawn, NPC repair FX, NPC damage
  notifications, `/giveallskills`, TargetManager iterator crash.
- Market: seeded all 30 NPC-station regions, parallelised seeding, `orderID` widened to BIGINT.
