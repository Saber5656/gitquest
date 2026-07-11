# Title

Blame reader: porcelain parser producing per-line author times

## Summary

Implement `git blame --porcelain` execution and parsing to answer "when was
this line last changed", used by the Skeleton detector to age TODO lines.

## Context

DESIGN §6.4.2: a TODO line is a "bone" iff its blame author-time is ≥ 365 days
old. Blame is the most expensive git query we make, so the API is
per-file-on-demand and the pipeline budgets calls (issue 19/21).

## Scope

`internal/gitio/blame.go` (+ tests):

```go
// LineAges returns author-time (unix) for each line of path at commitSHA.
// Index 0 = line 1. Length equals the file's line count at that commit.
func LineAges(ctx context.Context, r *Runner, commitSHA, path string) ([]int64, error)
```

## Detailed Requirements

1. Command: `git blame --porcelain -w <commitSHA> -- <path>` (`-w` ignores
   whitespace-only changes so reindents don't refresh a TODO's age).
   `LineAges` validates `path` BEFORE invoking git (DESIGN §10.2/§10.6):
   must be non-empty, relative, valid UTF-8, no `..` segment, no NUL —
   violations return a typed error without spawning a process.
2. Porcelain parsing rules (implement exactly; write against captured real
   fixtures in `testdata/` like issue 06):
   - Header line: `<sha> <orig-line> <final-line> [<num-lines>]` starts a group.
   - Metadata lines (`author-time <unix>`, `author-tz`, …) appear only the
     FIRST time a commit appears in the output; cache `author-time` per SHA
     and reuse for subsequent groups of the same SHA.
   - Content lines start with `\t` and advance the final-line counter.
3. Output array must be dense: every final line 1..N gets a timestamp; a gap
   (parser desync) returns a typed error rather than silently wrong ages.
4. Uncommitted-lines edge (`0000…` SHA) cannot occur because we blame at
   `HEAD`, not the worktree; assert defensively and treat all-zero SHA groups
   as `author-time = HeadTime` with a warning.
5. Timeout: rely on the runner default; a single blame > 120 s is a budget
   problem surfaced as a scan warning by the caller (21), not a crash.
6. Memory: O(lines) int64 slice; no retained porcelain text.

## Acceptance Criteria

- [ ] Fixture repo where line ages differ (3 commits touching different lines,
      fixed `GIT_AUTHOR_DATE`s) → exact expected int64 per line.
- [ ] Whitespace-only reindent commit does NOT change ages (`-w` proven by test).
- [ ] File whose lines were rearranged across commits parses densely without
      error (no claim about copy/move attribution — the command uses no `-C`/
      `-M` flags; ages reflect plain line-level blame).
- [ ] Path validation: absolute, `..`, empty, and NUL-containing paths →
      typed error, no subprocess spawned (spy on the runner).
- [ ] Corrupted porcelain (truncated fixture bytes) → typed parse error, no panic.
- [ ] Non-existent path → typed error mapped to "skip file with warning" by caller.

## Validation

```sh
go test -race ./internal/gitio/ -run TestLineAges
```

## Dependencies

- 04.

## Non-goals

- Bone/skeleton logic (21), blame budgeting (19/21), caching across scans (v2
  candidate per DESIGN U-2).

## Design References

- DESIGN §5.2, §6.4.2, §2.5 U-2.
