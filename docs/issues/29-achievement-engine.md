# Title

Achievement engine and the 15-achievement v1 set

## Summary

Implement the achievement evaluation pass: given pre/post scan facts, unlock
achievements exactly once and emit events, with the 15 fixed v1 definitions
from DESIGN §3.8.

## Scope

`internal/game/achievement.go` (+ tests):

```go
type ScanFacts struct { // everything achievements can see, assembled by scan (32)
    FirstScanCompleted bool
    SlaysThisScan      []Monster
    BossSlainThisScan  bool
    LifetimeSlays      int
    LifetimeDeletions  int64
    LifetimeCommits    int
    LivingMonsters     int
    LevelAfter         int
}
type Def struct {
    ID, Name, Hint string
    Unlocked       func(f ScanFacts, already map[string]Unlock) bool
}
func Defs() []Def // exactly the 15, in DESIGN §3.8 order
func Evaluate(f ScanFacts, already map[string]Unlock, now int64) (newUnlocks map[string]Unlock, events []Event)
```

## Detailed Requirements

1. The 15 definitions exactly per DESIGN §3.8 (ids `first-scan`,
   `first-blood`, six species `X-hunter/crusher/buster/splitter/breaker/
   toppler` names as listed, `boss-slayer`, `exterminator` (10 lifetime),
   `dungeon-cleared` (0 living AND ≥1 lifetime slay), `purifier` (≥10,000
   lifetime deletions), `centurion` (≥100 lifetime commits scanned),
   `deep-delver` (a slay this scan with `Depth(path) ≥ 6`), `level-10`
   (LevelAfter ≥ 10)).
2. `dungeon-cleared` uses LivingMonsters==0 && LifetimeSlays ≥ 1 — a repo with
   zero monsters on first scan does NOT unlock it (test).
3. Species-first achievements trigger on the species of any slay this scan;
   multiple species in one scan can unlock several at once.
4. Already-unlocked ids are never re-emitted (idempotence).
5. `Hint` strings (shown greyed in `achievements` command, 37) are part of
   the definitions here — write all 15 (e.g. exterminator: "Slay 10 monsters
   in this dungeon."). English, ≤ 60 chars each.
6. Event per unlock: `achievement_unlocked{id, name}`; order = Defs() order.
7. Pure; unlock timestamps from the `now` argument.

## Acceptance Criteria

- [ ] Table test per achievement: minimal ScanFacts that unlocks it, and a
      near-miss that doesn't (30 cases total) — this covers unlock BEHAVIOR;
      function fields are exercised, not compared.
- [ ] Multi-unlock scan (first scan + first blood + species + level-10) emits
      events in Defs() order.
- [ ] Idempotence across repeated Evaluate calls.
- [ ] Golden literal test on serializable metadata only: the 15 ids and names
      match DESIGN §3.8 exactly; the 15 hints match the literals defined in
      this issue (DESIGN does not define hint strings).
- [ ] `deep-delver` boundary: `Depth` counts path segments with root files at
      depth 1 (issue 14), so a slay at `a/b/c/d/e/f.go` (depth 6) unlocks and
      one at `a/b/c/d/e.go` (depth 5) does not. Both asserted.

## Validation

```sh
go test -race ./internal/game/ -run TestAchievement
```

## Dependencies

- 14, 26 (slays), 16 (LevelAfter), 15 (lifetime counters).

## Non-goals

- Rendering (37), persistence mechanics (10), achievement art.

## Design References

- DESIGN §3.8, §3.10.
