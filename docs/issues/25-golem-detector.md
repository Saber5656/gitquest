# Title

Golem detector: monolithic files

## Summary

Detect huge single files — by line count for readable text, by blob size for
content-skipped code files — per DESIGN §6.4.6 with the exact size-variant
constant fixed here.

## Scope

`internal/detect/golem.go` (+ tests).

Implements `Detector`. Config: `golem_min_lines` (default 1000, clamp
300–50000).

## Detailed Requirements

1. Text path: `ctx.Lines != nil && len(ctx.Lines) ≥ golem_min_lines` →
   HP = `min(floor(lines/20), 800)`; Evidence = `"<lines>-line monolith"`.
2. Size path (content-skipped blobs): `ctx.Truncated && ctx.Lang.Known &&
   ctx.Lang.CodeLike` → HP = `min(floor(size_bytes/1600), 800)` (DESIGN
   §6.4.6: one HP per 1,600 bytes ≈ 20 lines at 80 B/line, aligning the two
   paths). Evidence = `"~<MiB> MiB monolith (content skipped)"` (1 decimal).
   Rationale: for oversized blobs the pipeline never decoded content, so
   binary-ness is unknown; a 3 MiB `whatever.go` is a Golem regardless
   (`Truncated` is set only for over-budget blobs, never for binaries —
   FileContext contract in DESIGN §6.1 / issue 19).
4. Span nil; Fingerprint `"file"`.
5. Generated/vendored monoliths never arrive (18); Markdown/data files with
   `CodeLike=false` are eligible via the TEXT path only (a 5,000-line .md is
   a fair Golem) — the size path stays code-only to avoid flagging large
   data assets.

## Acceptance Criteria

- [ ] 1000-line file at default → HP 50; 999 lines → nothing; 16,000+ lines →
      HP 800 (cap).
- [ ] Truncated 3 MiB `.go` → size path: floor(3·1024·1024/1600) = 1966 →
      capped, golden HP = 800.
- [ ] Truncated 2.5 MiB `.csv` (not in langmap → `Known=false`) → no finding
      (size path is code-only).
- [ ] Binary blob → no finding.
- [ ] Markdown 5,000 lines → text-path Golem.
- [ ] Config `golem_min_lines=300` catches a 300-line file (HP 15).

## Validation

```sh
go test -race ./internal/detect/ -run TestGolem
```

## Dependencies

- 19 (FileContext.Lang already carries the langmap data from 17).

## Non-goals

- Function-length analysis (needs parsing — v2/never per ADR-005); archive
  content inspection.

## Design References

- DESIGN §6.4.6, §6.4.7, §9.3.
