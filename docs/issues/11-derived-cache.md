# Title

Derived cache store: per-file last-touched index

## Summary

Persist the rebuildable derived data (`cache.json`): the last-touched unix
timestamp per repo path, built from the full history stream on first scan and
updated incrementally, consumed by Ghost (22) and Skeleton (21).

## Context

DESIGN §5.3: without this cache, per-file age queries would cost
O(files × history). Cache corruption must never lose game progress — it is
rebuilt silently (DESIGN §7.3).

## Scope

`internal/state/cache.go` (+ tests):

```go
type Cache struct {
    SchemaVersion int               `json:"schema_version"` // 1
    BuiltFromSHA  string            `json:"built_from_sha"` // HEAD at build time
    LastTouched   map[string]int64  `json:"last_touched"`   // path → unix ts
}
func LoadCache(prof Profile) (*Cache, bool /*rebuilt*/, error)
func SaveCache(prof Profile, c *Cache) error // atomic like state, no .bak
func (c *Cache) ApplyCommit(when int64, paths []string)
func (c *Cache) ApplyRenames(renames map[string]string)
func (c *Cache) Prune(existing func(path string) bool)
```

## Detailed Requirements

1. `LoadCache`: parse failure or size > 256 MiB or schema mismatch ⇒ return
   empty cache with `rebuilt=true` (caller triggers full-history rebuild);
   never a fatal error (DESIGN §7.3).
2. `ApplyCommit` sets `max(existing, when)` for each path (out-of-order
   safety). Path-validity contract: callers pass only paths already validated
   by the gitio parsers (06: relative, no `..`, no NUL, valid UTF-8);
   `ApplyCommit`/`ApplyRenames` additionally DROP any path violating those
   rules defensively (silent for cache purposes — the gitio layer already
   warned) so hostile strings can never persist into `cache.json` (§10.2).
3. `ApplyRenames`: move timestamp old→new (keep newer if both exist), delete old.
4. `Prune` drops paths not in the current tree (bounds growth on long-lived
   repos); called by scan (32) after tree listing.
5. Atomic save via the shared `atomicWriteJSON` helper owned by issue 10
   (`internal/state`); issue 10 is therefore a hard dependency of this issue.
6. Deterministic marshal: byte-stable output across saves of equal content.
   Go's `encoding/json` already sorts string map keys — plain marshaling is
   acceptable; the requirement is the STABILITY (proven by test), not custom
   encoder code.
7. File permissions 0600 via `CreatePrivate` (issue 09).

## Acceptance Criteria

- [ ] Round-trip with 10k synthetic paths; sorted-key output verified stable
      across two saves (byte-identical).
- [ ] Corrupt cache.json → LoadCache returns empty+rebuilt, file replaced on
      next SaveCache; no error surfaces.
- [ ] ApplyCommit monotonicity: older timestamp does not overwrite newer.
- [ ] ApplyRenames + Prune behave per spec (table tests).
- [ ] Oversized cache (>256 MiB simulated via a size-checking seam, not a real
      file) → rebuilt path taken.

## Validation

```sh
go test -race ./internal/state/ -run TestCache
```

## Dependencies

- 09; 10 (shared `atomicWriteJSON`); 06 (history stream shape —
  `ApplyCommit` is fed from `WalkHistory`).

## Non-goals

- Deciding WHEN to rebuild (scan command, 32). Blame result caching (v2, U-2).

## Design References

- DESIGN §5.3, §7.1, §7.3.
