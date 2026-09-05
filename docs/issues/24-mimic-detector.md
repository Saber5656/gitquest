# Title

Mimic detector: committed merge-conflict markers

## Summary

Detect complete conflict-marker sets (`<<<<<<< ` … `=======` … `>>>>>>> `)
accidentally committed to tracked text files, per DESIGN §6.4.5.

## Scope

`internal/detect/mimic.go` (+ tests).

Implements `Detector`. No config keys.

## Detailed Requirements

1. State machine over lines (column-0 anchored, exact):
   - START: line starts with `<<<<<<< ` (7 `<` + space) → record start,
     go MID-WAIT;
   - MID-WAIT: line exactly `=======` → go END-WAIT; another `<<<<<<< ` →
     restart at new line; EOF → discard partial;
   - END-WAIT: line starts with `>>>>>>> ` → COMPLETE SET (record end line +
     both labels); `<<<<<<< ` → restart; EOF → discard.
   `|||||||` (diff3 base marker) is allowed inside MID-WAIT without effect.
2. Applicability: `ctx.Lines != nil` (text within budget). All languages
   including Markdown (DESIGN accepts docs-that-explain-conflicts as noise;
   tame exists).
3. One finding per file when ≥ 1 complete set:
   HP = `min(40 * sets, 200)`; Span = first set's `[start, end]`;
   Evidence = `"<sets> committed conflict marker set(s)"`;
   Fingerprint = first 16 hex of sha256 over joined `start:end:leftLabel:rightLabel`
   for all sets (labels = text after the marker, whitespace-trimmed,
   length-capped 64 chars each).
4. Labels go through the render sanitizer downstream; the detector stores raw
   (capped) text — no terminal bytes escape here (Evidence is static text).
5. Single pass, O(lines).

## Acceptance Criteria

- [ ] Complete set detected with exact span; two sets → HP 80, sets=2.
- [ ] Incomplete: `<<<<<<<` without `=======`; `=======` alone (markdown
      heading underline); `>>>>>>>` alone → no finding (3 fixtures).
- [ ] Indented markers (` <<<<<<< `) → no finding (column-0 rule).
- [ ] `=======` of length ≠ 7 (e.g. 20 `=` markdown underline) → not a
      separator (exact-match rule proven).
- [ ] diff3 `|||||||` inside a set doesn't break completion.
- [ ] Restart logic: `<<<<<<< a`, `<<<<<<< b`, `=======`, `>>>>>>> c` → one
      set anchored at the SECOND start line.
- [ ] Golden fingerprint; label with ANSI bytes is length-capped and stored,
      test asserts no `\x1b` reaches Evidence (Evidence is the static
      template).

## Validation

```sh
go test -race ./internal/detect/ -run TestMimic
```

## Dependencies

- 19.

## Non-goals

- `.gitattributes`/merge-driver awareness; binary conflict detection.

## Design References

- DESIGN §6.4.5, §6.4.7, §10.3.
