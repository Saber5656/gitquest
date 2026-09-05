# Title

`status` command: player card and dungeon overview

## Summary

Implement `gitquest status` — the at-a-glance read-only view: level, XP bar,
monster counts, boss line, active quests, and the last 5 events.

## Scope

`internal/cli/status.go` (+ golden tests).

Supports `--json` (subset schema: `player`, `boss`, `counts`, `quests`,
`recent_events` — field names identical to the report schema (39) where they
overlap).

## Detailed Requirements

1. Read-only: no lock, no scan, no state mutation (DESIGN §8.1 note). If no
   scan has ever run: print `Run 'gitquest scan' to generate this dungeon.`
   → exit 0.
2. Layout (mono golden fixes exact form; dark theme adds color only):
   ```
   ⚔ gitquest — <repo basename>
   Lv 11  [████████░░░░░░░░░░░░░░░░]  15,230 XP · 2,310 into · 1,810 to Lv 12
   Monsters: 23 alive (1 boss, 3 guardians) · 17 slain · 2 tamed · 1 fled
   Boss: Grubmaw the Forgotten (Ghost, Lv. 12) — src/legacy/api.js [240 HP]
   Quests (3): ☠ Slay Bonebag the Dusty (Skeleton) — pkg/util/compat.go [50 HP → 63 XP]
               …
   Recent: ★ Reached level 11   ☠ Slew Squishlet the Damp (Slime)  …
   ```
   Numbers from `game.Progress` (16); boss/guardian from stored state
   (recomputed values persisted by scan — status never recomputes).
3. Recent events: last 5 from the state ring, newest first, one line each
   (`Renderer.EventLine`).
4. Stale-scan hint: if `HEAD != state.last_scan.sha`, append stderr note
   `dungeon state is behind HEAD — run 'gitquest scan'` (stdout stays clean
   for scripting). The HEAD read is a single `rev-parse` through the
   hardened `gitio.Runner` — sanctioned by DESIGN §8.2's status row
   ("state, git (HEAD rev-parse only)"); this is NOT a re-scan.
5. Quest reward preview: `hp + round(0.25*hp)` XP, i.e. exactly
   `round(1.25*hp)` with math.Round half-away-from-zero — e.g. hp 50 → 63
   (the single formula shared with 28/32/36).
6. `--json`: exact field list frozen by a schema golden in tests.

## Acceptance Criteria

- [ ] Golden output (mono, 80 col) for: fresh-after-first-scan state, mid-game
      state (all sections populated), cleared dungeon (0 alive → celebratory
      line `The dungeon is clear. New corruption will spawn in time…`),
      never-scanned (hint line).
- [ ] Stale-HEAD note appears on stderr only when SHAs differ.
- [ ] `--json` golden validates; no repo-derived string bypasses Sanitize
      (fixture with hostile path in state).
- [ ] Runs without lock while a concurrent `scan` holds it (no contention).

## Validation

```sh
go test -race ./internal/cli/ -run TestStatus
```

## Dependencies

- 10, 16, 30, 31; 04/05 (runner + discovery for the HEAD rev-parse hint).

## Non-goals

- Re-scanning, map rendering (34), full report (39).

## Design References

- DESIGN §8.2 (status row), §3.10, §8.1.
