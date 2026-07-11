# Title

XP engine: commit folding, churn/purge bonuses, per-author ledger

## Summary

Implement the pure XP computation that folds a stream of commits into XP
deltas and lifetime counters, exactly per the DESIGN §3.4 formula table
(base 10, log-churn bonus cap 40, purge +5, merge flat 5).

## Context

XP is the core progression currency. Formulas are game balance — they get
golden-table tests so refactors can't silently change them (ISSUE_PLAN §6.2).
Slay bonuses are NOT computed here (reconciliation, 26, emits them).

## Scope

`internal/game/xp.go` (+ tests). `internal/game` imports only stdlib (DESIGN
§4.1), so the engine defines its own minimal input struct instead of consuming
`gitio.Commit`; the adapter from `gitio.Commit` to `CommitFact` (summing
numstat, hashing the author email via `state.HashAuthor`) lives in the scan
command wiring (issue 32):

```go
type CommitFact struct {
    IsMerge    bool
    Insertions int64 // sum over files; binary numstat entries contribute 0
    Deletions  int64
    AuthorKey  string // pre-hashed (sha256 hex of lowercased email)
    When       int64
}

type AuthorXP struct{ XP, Commits int }

type XPResult struct {
    Total      int
    Commits    int
    Insertions int64
    Deletions  int64
    PerAuthor  map[string]AuthorXP // keyed by CommitFact.AuthorKey
}

func CommitXP(f CommitFact) int
func FoldCommits(fs []CommitFact) XPResult
```

## Detailed Requirements

1. `CommitXP` exactly (DESIGN §3.4):
   - merge (`IsMerge`): return 5.
   - else: `10 + min(floor(10*log10(1+I+D)), 40) + purge` where
     `purge = 5` iff `D > I && D ≥ 10`, else 0. Use `math.Log10` on float64 of
     the int64 sum; values are ≥ 0 by contract (clamp negatives to 0
     defensively).
2. `FoldCommits` accumulates Total/Commits/Insertions/Deletions and per-author
   `{XP, Commits}` keyed by `AuthorKey`; empty AuthorKey buckets under `""`
   (rendered nowhere in v1, but persisted — I-5).
3. Binary numstat entries (I=-1/D=-1 from issue 06) are the ADAPTER's problem:
   spec for issue 32 says they contribute 0/0; note this contract in godoc here.
4. Overflow safety: XP int (platform int, ≥ 64-bit assumed via linux/darwin
   targets); totals int64 where sums of lines occur.
5. Pure functions, no time.Now, no I/O.

## Acceptance Criteria

- [ ] Golden table (≥ 12 rows) locked in a test with exact expected ints,
      including at least: (I=0,D=0) → 10; (I=99,D=0) → 30 (base 10 + churn 20);
      (I=0,D=10) → 25 (10 + churn 10 + purge 5); (I=500,D=600) → 45
      (10 + churn 30 + purge 5); (I=10⁹,D=10⁹) → 50 (10 + churn capped 40;
      no purge since D is not > I). Implementer verifies each row's
      arithmetic before freezing; goldens are authoritative once written.
- [ ] Merge commit with huge numstat → exactly 5.
- [ ] FoldCommits sums match a hand-computed 5-commit scenario incl. two
      authors.
- [ ] Property test: XP monotonically non-decreasing in D for fixed I.

## Validation

```sh
go test -race ./internal/game/ -run 'TestCommitXP|TestFoldCommits'
```

## Dependencies

- 14; 06 (shape of the adapter's source, contract only).

## Non-goals

- Level thresholds (16), slay bonuses (26), event emission (32).

## Design References

- DESIGN §3.4, §4.1 (purity rule), §7.2 (authors map), I-5.
