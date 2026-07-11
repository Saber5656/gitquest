# Title

`explore` views: Map and Bestiary (detail + tame confirmation)

## Summary

Implement the Map and Bestiary screens for `explore`: navigable dungeon tree,
monster list with detail pane, and the in-TUI tame flow reusing the CLI tame
code path, per DESIGN §8.5.

## Scope

`internal/tui/viewmap.go`, `viewbestiary.go` (+ model tests). Implements the
`ScreenView` interface from 41.

## Detailed Requirements

1. Map screen (list mode only, no detail pane of its own): scrollable tree
   identical in content to `map --all --depth 4` (34's construction logic
   MUST be shared — extract the tree builder into `internal/game` or
   `internal/render` when implementing 34; this issue consumes, never
   re-implements). `j/k` scroll; `enter` on a node emits
   `NavigateTo{Screen: bestiary, Mode: list, Preselect: <path prefix>}`
   (the shell API from 41 — Map never mutates Screen itself); the filter
   shows as a breadcrumb; `esc` in Bestiary list clears the filter before
   acting as no-op.
2. Bestiary screen:
   - List mode: rows like `monsters` (35) — name, species glyph, LV, HP,
     truncated room; filterable by the map jump; `j/k` move, `enter` detail.
     Status filter cycles with `f` (alive → all → tamed → alive…), shown in
     the footer.
   - Detail mode: full monster card — name+epithet, species lore line (one
     flavor sentence per species, fixed table written in this issue),
     path:span, HP/level, age, evidence, kill/tame history (`aka`, resolved),
     and the actions footer: `[t] tame  [esc] back` (t only when alive).
   - Confirm mode (`t`): modal `Tame <name>? It will never be reported
     again. [y/N]`; `y` → perform tame; `n`/`q`/`esc` → cancel.
3. Tame execution: acquire the profile lock transiently, re-load state, apply
   the SAME mutation function as CLI tame (38 — extract
   `state.ApplyTame(st, monsterID, reason)` shared helper), save, update the
   in-memory Model snapshot (monster status + counts), release lock. Lock
   busy → non-fatal toast message in the footer (`dungeon is busy — try
   again`), no crash. No reason input in TUI v1 (reason is CLI-only).
4. Footer toasts: 3-second transient messages (`Tamed Bonebag the Dusty`)
   via tea.Tick; cleared on any keypress. Glyph/emoji decoration follows the
   render layer's theme rules (31: mono → ASCII); toast copy itself is plain
   text.
5. All strings sanitized at render (31); long paths middle-truncated to pane
   width.
6. Performance: list virtualization for > 500 monsters (render window only).

## Acceptance Criteria

- [ ] Model tests: map→bestiary jump carries the path filter; breadcrumb
      shown; esc clears filter.
- [ ] Detail card golden (mono) for one monster of each species (6 goldens,
      lore lines included).
- [ ] Tame flow: y mutates state on disk (fixture assert) + updates list row
      to tamed + toast; n leaves everything untouched.
- [ ] Lock-busy path: toast, no mutation, TUI alive (inject held lock in test).
- [ ] `f` filter cycle order proven; footer reflects filter.
- [ ] 1,000-monster fixture renders the visible window only (
      render-call counting or timing bound in test).

## Validation

```sh
go test -race ./internal/tui/ -run 'TestMapView|TestBestiary|TestTameFlow'
```

## Dependencies

- 41, 34 (shared tree builder), 35 (row format parity), 38 (ApplyTame helper).

## Non-goals

- Quests/Status screens (43); editing reasons; rescan.

## Design References

- DESIGN §8.5, §3.3, §1.3 (tame as first-class).
