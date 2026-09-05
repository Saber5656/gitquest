# Title

State schema v1, atomic store, advisory locking, corruption recovery, migration framework

## Summary

Implement the authoritative `state.json`: typed schema (DESIGN §7.2), atomic
save with `.bak` rotation, exclusive advisory lock, recovery ladder, and a
versioned migration framework (v1 ships with version 1 only).

## Context

This is the single source of persistent game truth. Invariant I-5 requires the
schema to keep per-author extension possible. DESIGN §7.3 fixes durability
rules and exit code 4 semantics.

## Scope

`internal/state/schema.go`, `store.go`, `lock.go` (+ tests):

```go
type State struct {
    SchemaVersion  int               `json:"schema_version"`
    Repo           RepoInfo          `json:"repo"`
    Player         Player            `json:"player"`
    LastScan       *ScanMark         `json:"last_scan"`
    LastScanHealth ScanHealth        `json:"last_scan_health"` // warnings []string, skipped_files int, budget_hits []string (DESIGN §7.2/§8.3)
    QuestCounter   int               `json:"quest_counter"`    // monotonic quest-id source (issue 28)
    Monsters       []Monster         `json:"monsters"`
    Quests         []Quest           `json:"quests"`
    Achievements   map[string]Unlock `json:"achievements"`
    Events         []Event           `json:"events"`
    // raw retains unknown top-level fields for forward compatibility.
}
func NewState(repoID, rootSHA, repoPath string, now int64) *State

type Store struct{ /* prof Profile + repo metadata + clock */ }
// NewStore carries everything Load needs to mint a fresh State when no
// state.json exists yet.
func NewStore(prof Profile, repoID, rootSHA, repoPath string, clock func() int64) *Store
func (s *Store) Load() (*State, LoadHealth, error)   // recovery ladder inside
func (s *Store) Save(st *State) error                 // atomic, rotates .bak
func (s *Store) AcquireLock() (release func(), err error) // non-blocking
```

Field shapes: exactly DESIGN §7.2 (Monster with `id/aka/species/name/path/span/
hp/level/status/evidence/first_seen/resolved/tame_reason`; Player with
`xp/level/lifetime/authors` map keyed by sha256(lowercase email) hex; top-level
`quest_counter` int (monotonic quest-id source, issue 28); Events ring max 200;
Quests history max 50).

## Detailed Requirements

1. Atomic save: marshal (2-space indent) → write `state.json.tmp` → `fsync`
   file → close → rename current `state.json` to `state.json.bak` (if exists)
   → rename tmp over `state.json` → best-effort fsync of the directory.
   Any error leaves either the old or the new complete file in place, never a
   partial one at the final path.
2. Recovery ladder in `Load` (returns `LoadHealth ∈ {ok, recovered_bak, fresh}`):
   `state.json` parse ok → ok; else `.bak` parse ok → copy bak over state
   (via the atomic path), return recovered_bak; else if neither exists →
   fresh (`NewState`); else both exist and both corrupt → typed
   `ErrStateCorrupt` (CLI maps to exit 4 with the recovery message from
   DESIGN §7.3, mentioning `gitquest reset --hard` and what is lost).
3. Locking: create/open `lock` file, `syscall.Flock` `LOCK_EX|LOCK_NB` on
   unix; on Windows use a create-exclusive sentinel fallback (`O_CREATE|O_EXCL`
   with PID inside, stale if PID dead — best effort, documented). Busy →
   typed `ErrLockBusy` (CLI: exit 1, message
   `another gitquest is exploring this dungeon`).
4. Unknown-field preservation: unmarshal into `map[string]json.RawMessage`
   first, decode known keys into structs, keep unknown keys verbatim, re-emit
   them on save. Same one level deep inside `player` (cheap future-proofing;
   deeper levels not required — document).
5. Migration framework: `migrations = map[int]func(*rawState) error` applied
   in order while `schema_version < current`. v1 registers none. A state with
   `schema_version > current` → typed error "created by a newer gitquest"
   (exit 4 family, but distinct message; no auto-downgrade).
6. Size guards: refuse to load state.json > 64 MiB (typed error, corrupt
   ladder applies); events trimmed to 200 and quests history to 50 on save.
7. All timestamps int64 unix seconds; email hashing helper
   `HashAuthor(email string) string` = hex(sha256(lowercase(trim(email)))).

## Acceptance Criteria

- [ ] Round-trip: NewState → Save → Load equals (reflect.DeepEqual) with
      health ok.
- [ ] Crash-injection tests: kill the write between tmp-write and rename
      (simulate by calling internal steps) → Load still returns previous
      state; after bak rotation step → Load returns one of the two complete
      versions, never errors.
- [ ] Corrupt state + good bak → recovered_bak and state.json repaired on disk.
- [ ] Both corrupt → typed `ErrStateCorrupt` carrying the recovery message
      (the exit-4 CLI mapping itself is tested in issue 30, which owns
      `cli.ExitCode`).
- [ ] Unknown top-level field survives Save/Load byte-for-byte.
- [ ] schema_version 2 file → "newer gitquest" error; version 0 with a dummy
      registered migration in test → migrated to 1.
- [ ] Two processes (test spawns a child `go run` helper or two lock attempts
      in-process on separate fds) → second gets ErrLockBusy.
- [ ] `HashAuthor(" Foo@Bar.COM ")` == sha256 hex of `foo@bar.com`.

## Validation

```sh
go test -race ./internal/state/ -run 'TestStore|TestLock|TestMigrat|TestHashAuthor'
```

## Dependencies

- 09; 14 (Monster/Event/Quest type shapes — coordinate: 14 defines game types,
  this issue defines their JSON persistence; if 14 lands later, define the
  JSON structs here and 14 aliases them. Preferred order: 14 before or with 10).

## Non-goals

- Cache file (11). Reconciliation semantics (26). Reset UX (40).

## Design References

- DESIGN §7.2, §7.3, §11 (exit 4), I-5; ADR-004.
