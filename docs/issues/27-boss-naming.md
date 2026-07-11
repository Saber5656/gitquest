# Title

Boss/guardian selection and deterministic monster naming

## Summary

Implement dungeon-boss and floor-guardian selection over the living monster
set (DESIGN §3.7) and the deterministic fantasy-name generator (§3.9).

## Scope

`internal/game/boss.go`, `naming.go` (+ tests):

```go
// Boss returns the dungeon boss id ("" if no living monsters) and the
// per-floor guardian ids. Floor key = first path segment; root files use
// the internal key "" (rendered as "Entrance Hall" per DESIGN §3.2 — the
// render layer owns the label, this package owns the key).
func Boss(monsters []Monster) (bossID string, guardians map[string]string)
// BossEvents compares old and new boss ids and emits boss_appeared/boss_slain.
func BossEvents(oldBossID string, monsters []Monster, slainIDs []string, now int64) []Event

// NameFor is deterministic: "<Given> the <Epithet>". Given name derives from
// sha256(id); the epithet from (species, HP tier), so both are parameters.
func NameFor(id, species string, hp int) string
```

## Detailed Requirements

1. Boss = living monster with max HP; ties: earlier `FirstSeen.At`, then
   lexicographic smallest path, then id (total order — no ambiguity).
2. Guardians: for each floor (first path segment; root files → internal
   floor key `""`, rendered as `Entrance Hall` by the render layer), the
   max-HP living monster with HP ≥ 100 (same tie-breaking). The boss may
   simultaneously be its floor's guardian.
3. `BossEvents` logic: new boss ≠ old boss AND new ≠ "" → `boss_appeared`;
   old boss id ∈ slainIDs → `boss_slain` (in addition to its `monster_slain`
   from 26). Called on EVERY scan including the first — DESIGN §3.7 exempts
   the single boss_appeared from first-scan bulk suppression.
4. Naming (DESIGN §3.9): seed = sha256(id). Given name: 3 syllable tables
   (onset 16 × middle 16 × coda 16 = 4,096 minimum combinations) indexed by
   seed bytes; capitalized. Epithet: table lookup by (species, HP tier) where
   tiers are HP <50 / <150 / <400 / ≥400 → e.g. ghost tiers: "Pale",
   "Forgotten", "Ancient", "Eternal". 6 species × 4 tiers = 24 epithets, all
   distinct, all in the tables committed with this issue. Format:
   `<Given> the <Epithet>` (e.g. `Grubmaw the Forgotten`).
5. NameFor is pure and total (any `(id, species, hp)` → a name; unknown
   species falls back to a neutral epithet column, never panics); same inputs
   → same name on any platform (no locale dependence).
6. Display form with species/level (`"Grubmaw the Forgotten (Ghost, Lv. 4)"`)
   is a render concern (31) — NOT built here; NameFor returns name only.
7. HP tier boundaries and syllable tables are frozen by golden tests
   (changing them renames every monster — a game-breaking change; the test
   comment says so).

## Acceptance Criteria

- [ ] Boss selection table tests incl. all tie-breakers; empty set → "".
- [ ] Guardians: multi-floor fixture; HP-99 monster yields no guardian for
      its floor; root files land under internal floor key `""`.
- [ ] BossEvents: appearance on first boss, replacement (bigger monster
      appears), boss slain (both events fired across 26+27 integration
      fixture), no events when unchanged.
- [ ] NameFor goldens: 5 fixed `(id, species, hp)` tuples → 5 exact frozen
      names; distinctness sample: 1,000 sequential ids (same species/hp) →
      ≥ 950 distinct given names.
- [ ] 24 epithets present and distinct (reflection test).

## Validation

```sh
go test -race ./internal/game/ -run 'TestBoss|TestNaming'
```

## Dependencies

- 14, 26 (slainIDs contract).

## Non-goals

- Extra boss XP (none in v1 — DESIGN §3.7); boss art/ASCII (31/34).

## Design References

- DESIGN §3.7, §3.9, §3.10.
