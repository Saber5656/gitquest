# Title

Core domain types and the event model (`internal/game/types.go`)

## Summary

Define the pure domain vocabulary every other package shares: Monster, Player,
Quest, Achievement, ScanMark, and the closed Event set with payload shapes.
`internal/game` imports nothing but stdlib (DESIGN §4.1 dependency rule).

## Context

Types must exist before the engines (15–29) and persistence (10 aliases or
embeds them). DESIGN §3.3 (monster attributes), §3.10 (event set), §7.2 (JSON
shapes) are the authoritative field lists.

## Scope

`internal/game/types.go`, `events.go` (+ tests):

- `Species` string enum: `zombie skeleton ghost slime mimic golem` +
  `AllSpecies()` and `ValidSpecies(s)`.
- `MonsterStatus` enum: `alive slain fled tamed`.
- `Monster` struct exactly per DESIGN §7.2 (json tags included here so state
  (10) persists these structs directly): `ID string, Aka []string (cap 5),
  Species, Name, Path string, Span *[2]int, HP, Level int,
  Status MonsterStatus, Evidence string, FirstSeen Mark,
  Resolved *Resolution, TameReason string` (json `tame_reason`, omitempty).
- `Mark{At int64; ScanSHA string}`, `Resolution{Status MonsterStatus; At int64;
  Commit string}` (json `commit` serialized as `""` when unknown, e.g. tamed/fled).
- `MonsterLevel(hp int) int` = `clamp(hp/20+1, 1, 99)` (DESIGN §3.3).
- Explicit player structs (DESIGN §7.2):
  `Player{XP int (json xp); Level int; Lifetime Lifetime; Authors map[string]AuthorTotals (json authors)}`;
  `Lifetime{CommitsScanned int (json commits_scanned); Insertions, Deletions int64; Slays, Scans int}`;
  `AuthorTotals{XP, Commits int}`.
- `Quest{ID, MonsterID string, IssuedAt int64, Status QuestStatus}`
  (`active completed cancelled`), `Unlock{At int64}`.
- `Event{At int64; Type EventType; Data map[string]any}` with typed
  constructors for the CLOSED v1 set (DESIGN §3.10) and a machine-readable
  payload registry `EventSchemas map[EventType][]string` (sorted required
  Data keys per type) that tests and issues 32/33/39 consume. Payload keys
  (authoritative):
  | EventType | Data keys |
  |---|---|
  | `first_scan_completed` | `commits, xp, level, monsters` |
  | `xp_gained` | `xp, commits` (aggregated per scan) |
  | `level_up` | `from, to` |
  | `monster_appeared` | `monster_id, name, species, hp, path` |
  | `monster_slain` | `monster_id, name, species, hp, path, commit` |
  | `monster_fled` | `monster_id, name, species, path` |
  | `monster_tamed` | `monster_id, name, species, path` |
  | `quest_issued` | `quest_id, monster_id, name, reward_xp` |
  | `quest_completed` | `quest_id, monster_id, name, bonus_xp` |
  | `quest_cancelled` | `quest_id, monster_id, reason` (`fled`\|`tamed`) |
  | `achievement_unlocked` | `id, name` |
  | `boss_appeared` | `monster_id, name, hp` |
  | `boss_slain` | `monster_id, name` |
  Ad-hoc Event construction is forbidden by convention + a test that every
  EventType has exactly one constructor AND that each constructor's output
  keys equal `EventSchemas[type]`.
- `Depth(path string) int` — directory depth, root file = 1 (`a.go`→1,
  `x/a.go`→2).

## Detailed Requirements

1. Zero non-stdlib imports; a lint test asserts the import list.
2. All enums serialize as lowercase strings; unknown values fail decode with a
   typed error (state store surfaces it through the corruption ladder).
3. Constructors validate inputs (negative HP, empty ID → panic with message;
   these are programming errors, not runtime conditions).
4. `MonsterLevel` golden table: hp 0→1, 19→1, 20→2, 68→4, 1960→99, 5000→99.
5. `EventSchemas` is the machine-checkable payload contract (godoc points at
   it); issues 32/33 render from these payloads, issue 39 serializes them.

## Acceptance Criteria

- [ ] Enum round-trip JSON tests; invalid enum decode → typed error.
- [ ] Every EventType has a constructor whose produced Data keys exactly
      match `EventSchemas[type]` (test iterates the full EventType list —
      no reliance on godoc content).
- [ ] `MonsterLevel` and `Depth` golden tables pass.
- [ ] Import-purity test passes (stdlib only).

## Validation

```sh
go test -race ./internal/game/ -run 'TestTypes|TestEvents|TestMonsterLevel|TestDepth'
```

## Dependencies

- 01.

## Non-goals

- Any behavior (XP math, reconciliation, quests) — types only.
- Rendering/formatting of events (31/32).

## Design References

- DESIGN §3.3, §3.10, §7.2, §4.1 (import rule).
