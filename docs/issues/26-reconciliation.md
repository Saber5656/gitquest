# Title

Monster identity and reconciliation: slain/fled/tamed transitions, rename migration

## Summary

Implement the pure reconciliation engine that matches the previous scan's
monsters against fresh findings and produces status transitions, slay XP
bonuses, and events, per DESIGN §3.5 and §6.5.

## Context

This is where "delete dead code, get XP" actually happens, and where XP
farming via threshold toggling is prevented (fled ≠ slain). It must be pure
(`internal/game`) — inputs are data, no I/O.

## Scope

`internal/game/reconcile.go` (+ tests):

```go
type ReconcileInput struct {
    Now        int64
    ScanSHA    string
    Previous   []Monster            // from state (all statuses)
    Findings   []Finding            // from pipeline (converted: see below)
    TouchedPaths map[string]string  // path → last commit SHA touching it this scan range
    Renames    map[string]string    // old → new (from diff-tree); applied by Reconcile itself (req 2)
    FirstScan  bool
    // Naming fn (27), injected to keep files decoupled; needs species+hp
    // because epithets derive from (species, HP tier) per DESIGN §3.9.
    NameFor    func(id, species string, hp int) string
}
type Finding struct { // game-local mirror of detect.Finding (import purity)
    Species, Path, Evidence, Fingerprint string
    Span *[2]int
    HP int
}
type ReconcileResult struct {
    Monsters   []Monster // full new set (all statuses, retained history)
    SlayBonusXP int      // raw sum of slain HP; quest ×1.25 top-up is the scan command's job (req 6)
    Events     []Event
    Slays      []Monster // convenience: the slain ones this scan
}
func Reconcile(in ReconcileInput) ReconcileResult
```

(Adapter `detect.Finding` → `game.Finding` lives in scan wiring, issue 32.)

## Detailed Requirements

1. ID construction: `id = species + ":" + path + ":" + fingerprint` —
   computed here from Finding fields via the single authoritative function
   `MonsterID(species, path, fingerprint)`. The fingerprint is the literal
   `"file"` ONLY for the whole-file species ghost, slime, and golem;
   zombie, mimic, AND skeleton use the content fingerprints their detectors
   produce (DESIGN §6.5, §6.4.2).
2. Rename migration FIRST: for each previous monster whose path has an entry
   in `Renames`, rewrite its path (and id) before matching; record old id in
   `aka` (cap 5, FIFO). No event for pure moves.
3. Matching passes, in order:
   a. exact id equality;
   b. span-overlap fallback (span species only: zombie/mimic/skeleton): same
      species+path, `overlap(oldSpan, newSpan) ≥ 0.5 * len(oldSpan)` →
      match; id rewritten to the new fingerprint, old id appended to `aka`.
   Each finding matches at most one previous monster (greedy by largest
   overlap, ties by lower start line — deterministic).
4. Transitions (DESIGN §3.5):
   - matched + previous status `alive` → stays alive (HP/Span/Evidence
     refreshed from the finding; FirstSeen preserved).
   - matched + previous status `tamed` → stays tamed, finding consumed
     silently (no event, no XP — taming is permanent until untamed).
   - matched + previous `slain/fled` → RESURRECTION: status back to alive,
     `monster_appeared` event, FirstSeen updated to now, `Resolved` cleared
     (`aka` keeps ids only; a resurrection counter is not tracked in v1).
   - unmatched previous `alive`: if `TouchedPaths[path]` exists (or the path
     was deleted — deletion shows up in TouchedPaths from numstat) → `slain`
     (Resolution{slain, Now, commit=TouchedPaths[path]}), `monster_slain`
     event, slay bonus += HP; else → `fled` (Resolution{fled, Now, ""}),
     `monster_fled` event, no XP.
   - unmatched previous `slain/fled/tamed` → carried over unchanged.
   - unmatched finding → new monster: id, `NameFor(id, species, hp)`,
     level = `MonsterLevel(hp)`, status alive, FirstSeen{Now, ScanSHA},
     `monster_appeared` event — SUPPRESSED on `FirstScan` (DESIGN §3.5: bulk
     appearance replaced by the first-scan summary; monsters are still
     created).
5. Depth for events/report is derived (`Depth(path)`), not stored.
6. Quest multiplier is NOT applied here: `SlayBonusXP` is the raw HP sum;
   the quest engine (28) reports which slays complete quests and the scan
   command applies the ×1.25 top-up (keeps this engine quest-agnostic).
   Godoc states this split explicitly.
7. Determinism (authoritative ordering, encode in godoc + tests): output
   `Monsters` preserves the previous scan's order for retained monsters; new
   monsters are appended sorted by (Path, Species, Span.start). Events are
   grouped in the order slain → fled → appeared, each group sorted by HP
   descending. Rendering (32) relies on this ordering.
8. Pure function; no time, no I/O, no randomness.

## Acceptance Criteria

- [ ] Scenario tests (table-driven, ≥ 10 scenarios): survive, slay (touched),
      slay (file deleted), flee (untouched disappearance), tame persistence,
      resurrection, rename migration (id rewritten, aka recorded), span drift
      within 50% (same monster), span replaced < 50% overlap (old slain-or-
      fled + new appeared), first-scan suppression.
- [ ] Greedy overlap matching determinism: two candidate findings overlapping
      one old monster → larger overlap wins; equal → lower start line.
- [ ] SlayBonusXP equals sum of slain HP exactly; fled contributes 0.
- [ ] `aka` capped at 5 with FIFO eviction.
- [ ] Property test: every previous monster appears exactly once in output;
      every finding maps to exactly one output monster.
- [ ] Import-purity: `internal/game` still stdlib-only.

## Validation

```sh
go test -race ./internal/game/ -run TestReconcile
```

## Dependencies

- 14; consumes shapes from 19 (via adapter contract) and 06 (TouchedPaths/renames).

## Non-goals

- Naming table content (27), quest completion math (28), persistence (10).

## Design References

- DESIGN §3.5, §6.5, §3.10; §3.4 (slay bonus definition).
