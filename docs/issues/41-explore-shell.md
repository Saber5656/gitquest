# Title

`explore` TUI shell: bubbletea app, tabs, navigation state machine

## Summary

Implement the `explore` command's application shell: bubbletea program setup,
the 4-tab layout, the Screen×Mode state machine, global keybindings, terminal
safety (restore on panic, min-size guard), per DESIGN §8.5. View content
arrives in 42/43 — this issue ships with placeholder panes.

## Scope

`internal/tui/app.go`, `keys.go`, `internal/cli/explore.go` (+ model tests).

```go
type Screen int // ScreenMap, ScreenBestiary, ScreenQuests, ScreenStatus
type Mode int   // ModeList, ModeDetail, ModeConfirm
type Model struct {
    Screen Screen; Mode Mode
    State  *state.State // loaded once at startup (read snapshot)
    // per-screen sub-models registered by 42/43 via a ScreenView interface
}
type ScreenView interface {
    Update(msg tea.Msg, m *Model) (ScreenView, tea.Cmd)
    View(m *Model, width, height int) string
    // CanLeave reports whether tab-switching is allowed (false in detail/confirm).
}

// NavigateTo is the ONLY sanctioned cross-screen transition mechanism.
// A view returns it as a tea.Cmd message; the shell switches Screen/Mode,
// applies the preselect, and records returnTo so `esc` can go back.
// Used by 42 (map → bestiary jump) and 43 (quest detail → bestiary detail).
type NavigateTo struct {
    Screen    Screen
    Mode      Mode      // list or detail
    Preselect string    // monster id or path prefix filter
    ReturnTo  *NavState // non-nil: esc pops back to this screen/mode/selection
}
```

## Detailed Requirements

1. `explore` command bootstrap: same App wiring (30); loads state ONCE (no
   lock — read snapshot; taming in 42 acquires the lock transiently). Never-
   scanned → print the standard hint, exit 0, TUI not started.
2. Tab bar: `[1] Map · [2] Bestiary · [3] Quests · [4] Status` — active tab
   styled; keys `1`–`4`, `tab`, `shift+tab` switch ONLY in ModeList
   (DESIGN §8.5).
3. Global keys (all modes): `q`/`ctrl+c` quit (in ModeConfirm, `q` cancels
   the dialog instead — DESIGN §8.5); `?` toggles a help overlay listing
   keys (overlay is part of this issue).
4. State machine transitions (enforced in one `route(msg)` function,
   unit-tested exhaustively):
   `list --enter--> detail --esc--> list`;
   `detail --t--> confirm --y/n--> detail(updated)/detail` — the `t`
   transition exists ONLY when the Bestiary view is showing a living
   monster's detail (DESIGN §8.5); `t` anywhere else is a no-op;
   key-driven screen switches (`1-4`/tab) allowed only in list mode;
   view-initiated `NavigateTo` messages may switch screens from detail mode
   (the shell honors them regardless of Mode and maintains the ReturnTo
   stack, max depth 1); `esc` in list mode = no-op (unless a ReturnTo is
   pending, which pops it).
5. Terminal safety: bubbletea's alt-screen mode; explicit
   `defer program restore` around Run; panic in any view recovers, restores
   terminal, re-panics with context (bubbletea does most of this — verify and
   add a regression test with a deliberately panicking placeholder view).
6. Min size 80×20: smaller ⇒ full-screen "Window too small (need 80×20,
   have WxH)" instead of broken layout; live-updates on resize.
7. Placeholder panes render screen name + "arrives in issue 42/43" (removed
   by those issues).
8. Model unit tests use message-driven testing (no PTY): construct Model,
   feed `tea.KeyMsg`/`tea.WindowSizeMsg`, assert Screen/Mode/View strings.
9. Mouse: disabled in v1 (keyboard-only; document).

## Acceptance Criteria

- [ ] Transition table test covers every (Screen, Mode, key) combination in
      the route function (table generated exhaustively; illegal transitions
      are no-ops). NavigateTo handling: switch from any Mode, preselect
      applied, ReturnTo popped by esc, depth capped at 1.
- [ ] `q` quits from list/detail; `q` in confirm cancels (model test).
- [ ] Resize below/above threshold swaps the too-small screen in and out.
- [ ] Panicking view → terminal restored (teatest or documented manual check
      + recover-path unit test).
- [ ] Never-scanned repo → hint on stdout, exit 0, no alt-screen flicker.
- [ ] Help overlay lists all global + list-mode keys.

## Validation

```sh
go test -race ./internal/tui/ -run 'TestRoute|TestShell'
go build ./... && ./gitquest explore   # manual smoke on a fixture repo
```

## Dependencies

- 10, 30, 31.

## Non-goals

- Real view content (42/43); rescan-in-TUI; mouse support; animations.

## Design References

- DESIGN §8.5, ADR-003.
