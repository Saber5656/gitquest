# Title

Level curve and level-up computation

## Summary

Implement the level threshold function and helpers to convert XP totals to
levels and progress-to-next, per DESIGN §3.4's curve.

## Context

Used by scan (level-up events), status (XP bar), report (fields
`xp_into_level`, `xp_to_next_level`). Locked by goldens.

## Scope

`internal/game/level.go` (+ tests):

```go
func XPToReach(level int) int   // cumulative threshold; XPToReach(1)==0
func LevelForXP(xp int) int     // max L with xp >= XPToReach(L); cap 999
func Progress(xp int) (level, into, toNext int)
func LevelUps(beforeXP, afterXP int) []int // levels crossed, ascending
```

Input clamping (explicit, tested): `XPToReach(level ≤ 1)` returns 0;
`XPToReach(level > 999)` returns `XPToReach(999)`; `LevelForXP(xp < 0)` and
`Progress(xp < 0)` treat xp as 0; `LevelUps` with `afterXP ≤ beforeXP`
returns nil. No panics on any int input.

## Detailed Requirements

1. Formula (authoritative, matches DESIGN §3.4): per-term value is
   `floor(100 * k^1.5)` — multiply by 100 FIRST, then floor:
   `XPToReach(L) = Σ_{k=1}^{L-1} floor(100 * k^1.5)`. Compute terms with
   `math.Floor(100 * math.Pow(float64(k), 1.5))`; sum in int. Memoize the
   cumulative table internally (package-level slice built lazily under
   `sync.Once`, up to cap 999).
2. Frozen reference values (assert as literals): `XPToReach(2)==100`,
   `XPToReach(3)==382` (term k=2 is 282), `XPToReach(4)==901` (k=3: 519),
   `XPToReach(5)==1701` (k=4: 800), `XPToReach(10)==11102`. If implementation
   arithmetic disagrees with any literal, STOP and reconcile with DESIGN §3.4
   in the same PR — the formula and the goldens must agree before merge.
3. `Progress`: `into = xp - XPToReach(level)`, `toNext = XPToReach(level+1) -
   xp` (at cap 999: toNext = 0).
4. `LevelUps(150, 700)` with thresholds 100/382/… returns every crossed level
   in order (used by scan to emit one `level_up` event per level, DESIGN §3.10).
5. Pure, stdlib-only, deterministic.

## Acceptance Criteria

- [ ] Frozen golden table for L ∈ {1,2,3,5,10,20,50,100,999} (values computed
      by the formula, asserted as literals in the test file). DESIGN §3.4
      already lists the exact values for L2–L10; do NOT modify DESIGN — if
      computed values disagree with it, stop and reconcile before merging.
- [ ] Input clamping behavior verified (negative xp, level 0, level 1000).
- [ ] `LevelForXP(XPToReach(L)) == L` and `LevelForXP(XPToReach(L)-1) == L-1`
      property loop for L in 2..999.
- [ ] `LevelUps` returns empty when no threshold crossed; multi-level jump
      returns all levels.
- [ ] Concurrency: parallel `LevelForXP` calls race-clean (memo table).

## Validation

```sh
go test -race ./internal/game/ -run 'TestLevel|TestProgress|TestLevelUps'
```

## Dependencies

- 14, 15.

## Non-goals

- Event emission/wiring (32); rendering of XP bars (31/33).

## Design References

- DESIGN §3.4.
