# Title

Zombie detector: commented-out code blocks

## Summary

Detect runs of ≥ N consecutive line-comment lines whose content looks like
code (not prose), per DESIGN §6.4.1, producing `zombie` findings with
span-fingerprint identity.

## Context

The flagship heuristic. It must balance recall against the false positives
DESIGN explicitly accepts-but-mitigates (license headers, doc comments,
markdown prose). Thresholds are config keys (§9.3).

## Scope

`internal/detect/zombie.go` (+ fixture corpus in `internal/detect/testdata/zombie/`).

Implements `Detector` (issue 19). Config: `zombie_min_lines` (default 8,
clamp 3–200).

## Detailed Requirements

1. Applicability: `ctx.Lang.Known && len(ctx.Lang.LineMarkers) > 0 &&
   ctx.Lang.CodeLike && ctx.Lines != nil`.
2. Run detection: a line belongs to a run iff, after left-trimming
   whitespace, it starts with any line marker. Runs are maximal; a single
   non-comment line breaks the run. Marker text and following space are
   stripped to produce the "payload" for scoring.
3. Codeness score over the run's payload lines (DESIGN §6.4.1): fraction of
   lines matching ANY of:
   - ends with `;`, `{`, `}`, `)` or `):`
   - contains `=` (excluding `==`-only prose heuristic: require no
     surrounding alphabetic sentence — implement as: contains `=` AND NOT
     line matches `(?i)^[a-z ,.'"-]*=[a-z ,.'"-]*$`)
   - matches `\b(if|else|for|while|return|func|def|class|import|switch|case|try|catch|end)\b`
   Finding iff `run_lines ≥ zombie_min_lines` AND `codeness ≥ 0.4`.
4. Guards (each with a dedicated fixture):
   - runs beginning at line ≤ 20 containing `(?i)copyright|license|permission`
     → skip;
   - prose runs: > 50% of payload lines end with `.` or start with `#` → skip;
   - doc-comment guard (DESIGN §6.4.1): runs written with doc-comment
     syntax (`///`, `//!`, or a `/** … */` block opener on the first line)
     are skipped only when the run is ≥ 60% prose-like (prose-like line =
     ends with `.` or contains ≥ 4 space-separated lowercase words). `#`/`##`
     runs are handled by the separate prose guard above, not here. Python
     `"""` docstrings are NOT handled in v1 — line markers only (document);
   - shebang line never counts.
5. HP = `min(2 * run_lines, 500)`; Span = `[first, last]` (1-based, original
   file coordinates); Evidence = `"<run_lines>-line commented-out code block"`;
   Fingerprint = first 16 hex of sha256 over the payload with all whitespace
   collapsed to single spaces and lines joined by `\n`.
6. Multiple runs per file ⇒ multiple findings.
7. Performance: single pass, O(bytes); compiled regexes at package init.

## Acceptance Criteria

- [ ] Positive fixtures: Go block of 10 commented code lines; Python block
      with `#`; SQL with `--`; JS block at threshold exactly 8.
- [ ] Negative fixtures: 7-line block (below threshold); license header;
      prose comment paragraph (godoc style); markdown file (CodeLike=false);
      `.json` (no markers); doc-comment run (`///` rustdoc).
- [ ] Threshold override via config honored (min_lines=3 catches a 3-line block).
- [ ] Fingerprint stable under reindentation (whitespace collapse proven) and
      changes when payload text changes.
- [ ] Golden HP values for 3 fixtures.
- [ ] Corpus regression harness: directory of ≥ 12 fixtures with an
      `expected.json` (findings per file) — one go test drives all.

## Validation

```sh
go test -race ./internal/detect/ -run TestZombie
```

## Dependencies

- 19, 17.

## Non-goals

- Block-comment (`/* */`) scanning (v2); language docstrings; semantic
  "is this reachable" analysis (never — ADR-005).

## Design References

- DESIGN §6.4.1, §6.4.7, §9.3.
