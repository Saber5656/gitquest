# Title

Detector interface and scan pipeline orchestrator (budgets, worker pool)

## Summary

Implement the `Detector` interface (the v2 plugin boundary) and the pipeline
that builds the candidate set, reads blobs, fans out to a worker pool, applies
resource budgets, and returns findings + scan health, per DESIGN §6.1–§6.2.

## Context

This is the heart of monster generation and the main DoS surface (§10.5).
Species detectors (20–25) plug into it; scan (32) drives it.

## Scope

`internal/detect/detector.go`, `pipeline.go` (+ tests):

```go
// FileContext and Finding exactly as DESIGN §6.1 — FileContext carries Path,
// Size, Lines, IsBinary, Truncated, Depth, Lang, LastTouched, RepoNow,
// RepoAlive, LineAges; Finding carries Species, Path, Span, HP, Evidence,
// Fingerprint.
type Detector interface {
    Species() string
    Examine(ctx FileContext) []Finding
}

// AgeRequester (DESIGN §6.1): optional capability. When a detector implements
// it and WantsAges(ctx) returns true, the pipeline fills ctx.LineAges from
// the blame provider (within budget) BEFORE calling Examine. Examine stays
// pure — this is the only sanctioned extra-data channel.
type AgeRequester interface {
    WantsAges(ctx FileContext) bool
}

type Warning struct{ Code, Message string } // shared by detect (18 uses it too)

type AgesProvider func(path string) ([]int64, error) // wired to gitio.LineAges (08)

type PipelineInput struct {
    Entries     []gitio.TreeEntry       // pre-validated by ListTree (07)
    Blobs       *gitio.BlobReader
    Excluder    *Excluder
    Cfg         config.Config
    LastTouched func(path string) int64 // from cache (11); 0 = unknown
    RepoNow     int64
    RepoAlive   bool                    // propagated into every FileContext
    Ages        AgesProvider
}
type Health struct{ ScannedFiles, SkippedFiles int; Warnings []Warning; BudgetHits []string }
func RunPipeline(ctx context.Context, in PipelineInput, dets []Detector) ([]Finding, Health, error)
```

## Detailed Requirements

1. Candidate building order per file (entries arrive already path-validated
   by `ListTree` (07) — the pipeline defensively drops any entry that fails a
   re-check of the §10.2 rules, counting it in SkippedFiles):
   Kind==Blob → Excluder.ByPath → size ≤ 2 MiB (else: forward metadata-only
   FileContext with `Lines=nil, Truncated=true`; content-dependent detectors
   must no-op on nil Lines, while the metadata-only detectors Slime (23) and
   Golem (25) still examine such contexts) → blob read via 07
   (which implements the §6.2 binary heuristic — NUL in first 8 KiB or > 10%
   invalid UTF-8 in first 8 KiB — and the 64 KiB per-line truncation cap;
   the pipeline aggregates `BlobContent.LongLines` counts into one Health
   warning `long-lines-truncated: N files`) → IsBinary ⇒ metadata-only
   context with `Truncated=false` (Slime (23) must still see it) →
   `GeneratedHeader` (18) ⇒ excluded. `Truncated` is set ONLY for over-budget
   blobs, never for binaries (FileContext contract, DESIGN §6.1).
2. Budgets enforced exactly (DESIGN §6.2, config-clamped §9.3): max candidate
   files (default 100k — counted AFTER path exclusion), total decoded bytes
   (512 MiB), whole-pipeline deadline derived from the scan's 10-min context.
   On budget hit: stop admitting new files, record `BudgetHits` entry
   (`files`, `bytes`, or `deadline`) and per-file skip warnings aggregate into
   counts, never one warning per file.
3. Blame budget: for each detector implementing `AgeRequester`, the pipeline
   calls `WantsAges(ctx)` (cheap prematch, e.g. Skeleton's TODO regex — 21);
   if true and the distinct-blamed-file count is under
   `skeleton_blame_budget` (default 500), it invokes `Ages(path)` and sets
   `ctx.LineAges` before `Examine`. Over budget: `LineAges` stays nil, a
   single `blame` BudgetHit is recorded, and skipped-file counts aggregate
   into one warning. `Ages` errors → warning + nil LineAges (detector sees
   "no data", never an error).
4. Concurrency: blob reads are serial (BlobReader contract, 07); decode +
   Examine fan out to `min(GOMAXPROCS, 8)` workers via a bounded channel;
   findings collected via a mutex-guarded slice; result order normalized
   (sort by Path, Species, Span) for determinism.
5. Detector isolation: a panicking detector is recovered per file; the file
   gets a warning (`detector <species> panicked on <path>`), other detectors'
   findings for that file survive. A test detector panics deliberately.
6. `Cfg.Monsters.Disable` (`[monsters] disable` in TOML, §9.2): pipeline
   filters the detector list before running (unknown names already warned
   by 12). Ghost gating: `RepoAlive` is copied into every FileContext; the
   Ghost detector itself checks `ctx.RepoAlive` (22) — the pipeline does NOT
   special-case species beyond the disable list.
7. Findings validation before return: Path non-empty and present in the
   candidate set; HP ≥ 1 (0-HP findings dropped with warning — detector bug);
   Span nil or `1 ≤ start ≤ end ≤ len(lines)`; Fingerprint non-empty.
   Violations are dropped + warned (defense against future plugin detectors —
   this is the plugin boundary's contract enforcement point).
8. No detector may perform I/O: enforced by interface design (pure `Examine`)
   and documented; the ONLY sanctioned side channel is `Ages`.

## Acceptance Criteria

- [ ] Fixture pipeline run over a synthetic tree (mixed: excluded, binary,
      oversized, generated, normal) with two fake detectors → exact expected
      findings + Health counters (table-driven).
- [ ] Budget tests: file-count, byte, and blame budgets each trigger with
      correct BudgetHits and partial results (not errors).
- [ ] Panic-isolation test passes; `-race` clean under parallel Examine.
- [ ] Deterministic output ordering across 10 runs with GOMAXPROCS=8.
- [ ] Zero-HP finding from a fake detector is dropped with a warning.
- [ ] Cancelled context (deadline) → returns partial findings + `deadline`
      BudgetHit + no goroutine leak.

## Validation

```sh
go test -race -count=3 ./internal/detect/ -run TestPipeline
```

## Dependencies

- 07, 12, 14, 17, 18 (types/inputs); 08 (Ages signature).

## Non-goals

- Species logic (20–25); reconciliation (26); git plumbing (04–08).

## Design References

- DESIGN §6.1, §6.2, §9.3, §10.5; ADR-005 (plugin boundary).
