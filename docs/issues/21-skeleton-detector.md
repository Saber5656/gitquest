# Title

Skeleton detector: fossilized TODO/FIXME/HACK/XXX lines

## Summary

Detect files containing TODO-family markers whose blame age exceeds the
threshold; aggregate per file into one Skeleton whose bones are the stale
marker lines, per DESIGN §6.4.2.

## Context

The only detector needing line ages (blame), hence budgeted (issue 19's
`Ages` provider + `skeleton_blame_budget`). One Skeleton per file keeps
monster counts sane on TODO-heavy codebases.

## Scope

`internal/detect/skeleton.go` (+ fixtures).

Implements `Detector`. Config: `skeleton_min_age_days` (default 365, clamp
30–3650).

## Detailed Requirements

1. Implements `AgeRequester` (DESIGN §6.1 / issue 19): `WantsAges(ctx)` is
   the cheap prematch — scan `ctx.Lines` for regex
   `\b(TODO|FIXME|HACK|XXX)\b` (case-sensitive, word-bounded); return true
   iff ≥ 1 match. Zero matches ⇒ no blame call, no finding. The pipeline's
   blame budget counts only files where `WantsAges` returned true.
2. `Examine` reads `ctx.LineAges`; when nil (budget exhausted, blame error,
   or WantsAges false) ⇒ return no findings (the pipeline already recorded
   the warning). Examine performs no I/O.
3. Bones: matched lines where
   `floor((ctx.RepoNow - ctx.LineAges[line-1]) / 86400) ≥
   skeleton_min_age_days`. Zero bones ⇒ no finding (young TODOs are fine).
4. HP = `min(15*bones + 10*floor(max_bone_age_days/365), 400)`;
   Span = `[first_bone_line, last_bone_line]`; Evidence =
   `"<bones> fossilized TODO(s), oldest <years>y"` (1 decimal, e.g. `2.4y`);
   Fingerprint = first 16 hex of sha256 over the sorted, whitespace-collapsed
   bone line texts joined by `\n`.
5. Marker inside string literals/URLs counts (accepted noise, tame exists);
   but lines longer than 512 chars are skipped from bone candidacy (minified
   or data lines — cheap guard).
6. Deterministic: bones sorted by line number; identical content on different
   lines still distinct bones (line list drives span; fingerprint uses texts —
   collision between different bone SETS with equal texts is acceptable,
   §6.5's 50%-overlap rule disambiguates).

## Acceptance Criteria

- [ ] Fixture repo (real git, fixed dates): file with 2 stale TODOs (400d,
      800d) + 1 fresh (10d) → one Skeleton, bones=2, HP=15*2+10*2=50,
      correct span/evidence.
- [ ] File with only fresh TODOs → no finding; `WantsAges` returned true
      (blame consulted, counter=1 in the pipeline test).
- [ ] File without markers → `WantsAges` false (ages never populated; spy on
      the provider in the pipeline-level test).
- [ ] `ctx.LineAges == nil` (budget exhausted) → no finding, no panic.
- [ ] 600-char line containing TODO → not a bone.
- [ ] Golden fingerprint stability test.

## Validation

```sh
go test -race ./internal/detect/ -run TestSkeleton
```

## Dependencies

- 19, 08, 11 (ages plumbed from blame; RepoNow/LastTouched from cache—via pipeline).

## Non-goals

- Cross-scan blame caching (v2, U-2); issue-tracker link parsing; severity by
  marker type (TODO == FIXME in v1).

## Design References

- DESIGN §6.4.2, §6.4.7, §9.3, §2.5 U-2.
