# Title

Slime detector: backup and leftover artifact files

## Summary

Detect tracked files whose NAMES mark them as editor/merge/copy leftovers
(`*.bak`, `*.orig`, `*~`, "copy of", conflicted copies…), per DESIGN §6.4.4.
Content is never read — Slime applies to binary and oversized blobs too.

## Scope

`internal/detect/slime.go` (+ tests).

Implements `Detector`. No config keys.

## Detailed Requirements

1. Pattern set over the BASENAME, case-insensitive (DESIGN §6.4.4):
   suffixes `.bak`, `.orig`, `.old`, `.rej`, `.save`, `.swp`, `.tmp`, and
   trailing `~`; stem patterns `copy of *`, `* copy.*` (and `* copy` no-ext),
   `*_old.*`, `*_backup.*`, substring `(conflicted copy` (Dropbox style).
   Implement as an ordered list of (name, matcher) pairs — first match wins
   and its name goes into Evidence (`"leftover artifact (\*.bak)"`).
2. Applies to every `FileContext` the pipeline routes (the pipeline only ever
   builds contexts for tracked blob entries — symlinks/submodules never reach
   detectors, DESIGN §10.4), regardless of `IsBinary`/`Truncated`/nil
   `Lines`. Slime and Golem are the two nil-Lines-capable detectors
   (cross-check issue 19 requirement 1).
3. HP = `min(10 + floor(size_bytes/1024), 100)`; Span nil; Fingerprint `"file"`.
4. Guard: directories named like patterns are irrelevant (matching is on the
   basename of a blob path only). `.tmp`/`.save` INSIDE `testdata/**` never
   reach the detector (default excludes, 18) — no special handling here.
5. O(1) per file; no regex where a suffix check suffices.

## Acceptance Criteria

- [ ] Table test over ≥ 14 names: `api.go.orig`, `Notes.OLD`, `main.c~`,
      `debug.log.tmp`, `Copy of report.md`, `schema copy.sql`,
      `config_backup.yaml`, `x (conflicted copy 2024-01-02).txt` → Slime;
      `history.md`, `gold.txt`, `template.tmpl`, `save.go`, `oldman.go`,
      `backup/keeper.go` (basename `keeper.go`) → NOT Slime.
- [ ] Binary `.bak` (nil Lines) still detected; 3 MiB truncated `.orig` still
      detected with size-based HP.
- [ ] HP goldens: 0-byte → 10; 51200-byte → 60; 10 MiB → 100 (cap).
- [ ] Evidence names the matched pattern.

## Validation

```sh
go test -race ./internal/detect/ -run TestSlime
```

## Dependencies

- 19.

## Non-goals

- Content-based duplicate detection (v2 idea); untracked files (never seen —
  we scan the HEAD tree only).

## Design References

- DESIGN §6.4.4, §6.4.7.
