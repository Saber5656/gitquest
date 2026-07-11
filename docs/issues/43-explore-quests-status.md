# Title

`explore` views: Quests and Status

## Summary

Implement the remaining two `explore` screens: the quest board (list +
detail) and the status/trophies pane, completing the v1 TUI, per DESIGN §8.5.

## Scope

`internal/tui/viewquests.go`, `viewstatus.go` (+ model tests). Implements
`ScreenView` (41).

## Detailed Requirements

1. Quests screen:
   - List mode: one row per active quest (up to 3): target name, species
     glyph, room (truncated), reward preview (`62 XP`). Empty board shows the
     same copy as CLI (36).
   - Detail mode (`enter`): the full bounty card (36's layout adapted to the
     pane, including the Hint command line); `enter` on the detail's target
     emits `NavigateTo{Screen: bestiary, Mode: detail, Preselect: <monster
     id>, ReturnTo: quests/detail}` — the shell mechanism defined in 41.
     `esc` from the jumped-to bestiary detail pops back to quests/detail;
     `esc` again → quests/list. No route() bypass, no undocumented edges.
2. Status screen (single pane; formally always `ModeList` in the 41 state
   machine — `enter`/`esc`/`t` are no-ops here; `j/k` scroll the events
   region only):
   - Player card identical in content to CLI `status` (33): level, XP bar,
     counts, boss line.
   - Trophy case: the 15 achievements in a 3×5 grid (unlocked bright with
     date, locked dim with hint) — grid layout is TUI-specific.
   - Recent events: last 10 from the ring, scrollable with `j/k` when
     focused; `tab` between card/trophies/events sub-focus is NOT needed —
     single scroll region for events only (keep it simple).
3. Both screens read the Model's state snapshot; they update after a tame in
   42 (shared snapshot already mutated — verify counts refresh).
4. Sanitization + width discipline as everywhere (31).

## Acceptance Criteria

- [ ] Quest list/detail goldens (mono): 3-quest board, empty board, cleared
      dungeon.
- [ ] Cross-jump test: quests/detail → enter → bestiary/detail shows the
      right monster; esc returns to quests/detail; esc again → quests/list.
- [ ] Status pane golden: mid-game fixture (trophies 7/15, events populated);
      trophy grid aligns at 80 cols exactly.
- [ ] Event region scrolls; other regions fixed.
- [ ] After a tame in the same session (42's flow), status counts reflect it
      without restart (shared-model test).

## Validation

```sh
go test -race ./internal/tui/ -run 'TestQuestsView|TestStatusView'
```

## Dependencies

- 41, 42 (shared model + jump target), 36/37 content parity, 33.

## Non-goals

- Achievement unlock animations; quest mutation; multi-pane splits.

## Design References

- DESIGN §8.5, §3.6, §3.8.
