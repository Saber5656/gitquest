# Title

Quest engine: standing bounties over living monsters

## Summary

Implement quest slot management (max 3 active), refill selection, completion
and cancellation detection, per DESIGN §3.6.

## Scope

`internal/game/quest.go` (+ tests):

```go
type QuestUpdate struct {
    Quests      []Quest  // full new quest list (history capped at 50 by store)
    Completed   []Quest  // completed this scan (target slain)
    Cancelled   []Quest  // cancelled this scan (target fled/tamed)
    BonusXP     int      // Σ round(0.25 * hp) top-ups for completed quests
    Events      []Event  // quest_completed, quest_cancelled, quest_issued (this order)
}
// slain carries the full slain Monster values (HP as of slay time) so the
// bonus is computed from unambiguous data; fledOrTamed carries ids only.
func UpdateQuests(prev []Quest, monsters []Monster, slain []Monster,
                  fledOrTamed []string, now int64, nextID func() string) QuestUpdate
```

## Detailed Requirements

1. Resolution pass first: active quests whose `MonsterID` matches a Monster
   in `slain` → `completed` (+ event, + bonus `round(0.25*hp)` using THAT
   slain Monster's HP; combined with the base slay bonus from 26 this yields
   exactly `round(1.25*hp)` total per DESIGN §3.4 — hp + round(0.25·hp)
   equals round(1.25·hp) for integer hp under half-away-from-zero rounding).
   Active quests whose target id ∈ fledOrTamed → `cancelled` (+ event).
2. Refill pass: while active count < 3, pick from living monsters
   (status alive): order by HP desc; skip monsters already targeted by an
   active quest; at most ONE active quest per species at any time
   (DESIGN §3.6 "at most one per species"); skip tamed (not alive anyway).
   Each pick → new Quest{ID: nextID(), Status: active, IssuedAt: now} +
   `quest_issued` event.
3. `nextID` is injected: the scan wiring (32) provides a `q-%06d` formatter
   backed by the top-level `"quest_counter"` int field in state.json
   (DESIGN §7.2; the field is part of issue 10's typed schema). The engine
   itself never touches persistence.
4. Quests persist across scans until resolved (DESIGN §3.6): survivors stay
   active untouched (no re-issue events).
5. Rounding: `math.Round` semantics for the 0.25 bonus (test 0.5 boundary:
   hp=2 → bonus 1 (0.5 rounds half away from zero — Go's math.Round)).
6. Events order inside the update: completed → cancelled → issued.
7. Pure function; determinism via input order + explicit sorts.

## Acceptance Criteria

- [ ] Refill from empty: 3 quests issued, species-distinct, HP-descending
      preference proven with a fixture where a 2nd zombie outranks a slime
      (slime still picked for slot diversity).
- [ ] Completion: slain target → completed + bonus round(0.25·HP) + refill in
      the same update.
- [ ] Cancellation on flee and on tame.
- [ ] Species-uniqueness invariant holds across refills (property test over
      random monster sets, 1,000 iterations).
- [ ] Fewer than 3 living monsters → fewer quests, no padding, no dupes.
- [ ] `nextID` is called exactly once per issued quest, and returned ids are
      assigned in issue order (spy counter test — persistence of the counter
      itself is scan/state wiring, 32/10).

## Validation

```sh
go test -race ./internal/game/ -run TestQuest
```

## Dependencies

- 14, 26 (slain/fled sets).

## Non-goals

- Quest UI/copy (36), acceptance flow (none in v1), reward preview rendering.

## Design References

- DESIGN §3.6, §3.4 (×1.25 semantics), §3.10.
