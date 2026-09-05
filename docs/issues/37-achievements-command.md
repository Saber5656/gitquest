# Title

`achievements` command: trophy case

## Summary

Implement `gitquest achievements` — the 15 achievement slots with unlocked
timestamps and greyed hints for locked ones, per DESIGN §8.2.

## Scope

`internal/cli/achievements.go` (+ golden tests). Flag: `--json`.

## Detailed Requirements

1. Read-only, no lock; never-scanned hint as in status.
2. Layout: one line per achievement in `game.Defs()` order (29):
   - unlocked: `🏆 First Blood — unlocked 2026-07-02` (date from Unlock.At,
     UTC, `2006-01-02`);
   - locked: muted `🔒 Exterminator — Slay 10 monsters in this dungeon.`
     (the Hint string from 29).
   Header: `Trophies: 7 / 15`.
3. Mono glyph fallbacks `[x]` / `[ ]` (31's fixed mapping).
4. `--json`: `[{id, name, unlocked, unlocked_at?, hint}]` — full list, locked
   included (hint always present; report (39) includes only unlocked — note
   the deliberate difference: the trophy case teases, the report records).
5. No state mutation ever.

## Acceptance Criteria

- [ ] Goldens: fresh (1/15 after first scan), mid-game, complete (15/15 adds
      a final flourish line `Your legend is complete. For now.`).
- [ ] Order matches `Defs()` exactly (test iterates).
- [ ] Date formatting UTC-stable regardless of local TZ (set TZ in test).
- [ ] `--json` schema golden; unlocked_at omitted (not null) when locked.

## Validation

```sh
go test -race ./internal/cli/ -run TestAchievements
```

## Dependencies

- 10, 29, 30, 31.

## Non-goals

- New achievements (v2), unlock animations (TUI, 43).

## Design References

- DESIGN §3.8, §8.2 (achievements row).
