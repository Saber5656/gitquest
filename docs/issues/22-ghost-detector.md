# Title

Ghost detector: long-abandoned large files in a living repository

## Summary

Detect files untouched for ≥ 2 years but ≥ 200 lines, only when the repo
itself is active, per DESIGN §6.4.3. Ages come from the last-touched cache —
no git calls.

## Scope

`internal/detect/ghost.go` (+ fixtures).

Implements `Detector`. Config: `ghost_min_age_days` (default 730, clamp
90–3650), `ghost_min_lines` (default 200, clamp 50–10000).

## Detailed Requirements

1. Global gate: `ctx.RepoAlive == false` ⇒ `Examine` returns nothing
   (`RepoAlive` is a FileContext field per DESIGN §6.1, set by the pipeline
   from scan's computation "≥ 1 commit with `When ≥ RepoNow - 90d`").
   DESIGN: a dead repo is a mausoleum, not a haunted dungeon.
2. Finding iff: `ctx.Lines != nil` (text, within size budget) AND
   `ctx.LastTouched > 0` (unknown age ⇒ skip, never guess) AND
   `floor((ctx.RepoNow - ctx.LastTouched) / 86400) ≥ ghost_min_age_days`
   (this IS `age_days(LastTouched)` per DESIGN §6.4's definition —
   RepoNow-relative, subtract once) AND `len(Lines) ≥ ghost_min_lines`.
3. HP = `min(floor(lines/10) * clamp(floor(age_days/365), 1, 5), 600)`
   (DESIGN §6.4.3). The age factor is clamped to ≥ 1 so that a config-lowered
   `ghost_min_age_days` (clamp minimum 90 d < 365 d) can never yield a 0-HP
   finding, which the pipeline would drop as a detector bug (19 §7).
4. Span = nil (whole file); Evidence = `"untouched for <years>y, <lines> lines"`
   (years 1 decimal); Fingerprint = `"file"`.
5. No per-file git calls; O(1) beyond line count already available.

## Acceptance Criteria

- [ ] Fixture: 250-line file last touched 3 years ago in an active repo →
      HP = 25*3 = 75, correct evidence.
- [ ] Repo inactive (no commit in 90 d) → zero Ghost findings repo-wide.
- [ ] 150-line ancient file → no finding (below min_lines).
- [ ] `LastTouched == 0` → no finding.
- [ ] Config `ghost_min_age_days=90`, file aged 100 d → age factor clamps to
      1, HP = floor(lines/10), finding survives the pipeline (no 0-HP drop).

## Validation

```sh
go test -race ./internal/detect/ -run TestGhost
```

## Dependencies

- 19, 11 (LastTouched via pipeline input).

## Non-goals

- Import/reference analysis ("is it still used") — ADR-005; directory-level
  ghosts (v2 idea).

## Design References

- DESIGN §6.4.3, §6.4.7, §9.3.
