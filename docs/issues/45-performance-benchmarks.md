# Title

Performance benchmarks and budget enforcement tests

## Summary

Create the synthetic large-repo generator, `go test -bench` benchmarks for
the scan pipeline, and CI perf gates enforcing DESIGN §12's targets
(first scan ≤ 30 s @ 10k files/50k commits; incremental ≤ 3 s; read commands
≤ 150 ms; RSS ≤ 512 MiB).

## Scope

`internal/perf/` (build tag `perf`), `Makefile` target `bench`,
CI job `perf` (scheduled + on-demand label, NOT on every PR — runtime).

## Detailed Requirements

1. Synthetic generator (extends 44's builder): parameterized
   `GenRepo(files, commits, monsterDensity)` — creates a repo with
   ~10k files across realistic depth (mixture of langs from the langmap),
   ~50k commits via `git fast-import` stream writing (NOT 50k `git commit`
   invocations). Clarification: DESIGN §5.1's "all git access through
   gitio.Runner" invariant governs the PRODUCT binary; test-only fixture
   GENERATION under the `perf`/`e2e` build tags is outside that boundary.
   `fast-import` must never appear in `cmd/` or non-test `internal/` code
   (the depcheck of 46 asserts this). ~5% monster-bearing files of each
   species.
2. Benchmarks (`go test -bench`, b.ReportMetric for wall/RSS):
   B1 first scan; B2 incremental scan (100 new commits, 300 changed files);
   B3 `status`; B4 `monsters`; B5 pipeline-only (detectors over the tree,
   no git); B6 blame budget path (500 skeleton files).
3. RSS measurement targets the CHILD process (the real binary), not the test
   harness: linux — poll `/proc/<child-pid>/status` VmHWM every 100 ms while
   the child runs; darwin — read `syscall.Getrusage(RUSAGE_CHILDREN)` MaxRSS
   after the child exits (single-child harness so the attribution is exact).
   Record which method produced each number in baseline.json metadata.
4. CI gates (soft, per DESIGN §12): compare against committed baseline JSON
   (`internal/perf/baseline.json`); regression > +50% wall or > +25% RSS
   fails the job; baseline updated deliberately via `make bench-baseline`
   (PR reviewer sees the diff).
5. The 10-minute whole-scan timeout and byte/file budgets are asserted to
   trigger correctly on a pathological generated repo (200k files → budget
   skip warnings, clean exit 0 — DoS resistance proof for §10.5).
6. Document reproduction commands in `docs/perf.md` (created here): how to
   run locally, interpret metrics, update baselines.

## Acceptance Criteria

- [ ] B1 meets ≤ 30 s and ≤ 512 MiB on CI ubuntu-latest hardware (record
      actual numbers in baseline.json with the PR).
- [ ] B2 ≤ 3 s; B3/B4 ≤ 150 ms.
- [ ] Pathological-repo test exits 0 with budget warnings (never OOM/hang).
- [ ] Baseline mechanism proven: an intentional 2× slowdown commit (scratch)
      fails the perf job; reverted.
- [ ] `docs/perf.md` present with runbook.

## Validation

```sh
make bench          # local, prints table vs baseline
```

## Dependencies

- 32, 44 (builder/normalizer), 19 (budgets).

## Non-goals

- Micro-optimizations themselves (follow-ups if gates fail); Windows perf;
  profiling automation (pprof flags exist via Go toolchain already).

## Design References

- DESIGN §12, §10.5, §2.5 U-2/U-5.
