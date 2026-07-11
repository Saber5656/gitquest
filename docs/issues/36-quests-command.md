# Title

`quests` command: active bounty cards

## Summary

Implement `gitquest quests` — display up to 3 active quests as bounty cards
with target, reward preview, and an actionable hint, per DESIGN §8.2.

## Scope

`internal/cli/quests.go` (+ golden tests). Flag: `--json`.

## Detailed Requirements

1. Read-only, no lock; never-scanned hint as in status.
2. Card layout per active quest (mono golden authoritative):
   ```
   ☠ BOUNTY: Bonebag the Dusty (Skeleton, Lv. 3)
     Room:   pkg/util/compat.go:41-77
     Crime:  4 fossilized TODO(s), oldest 2.4y
     Reward: 50 HP → 63 XP (25% bounty bonus)
     Hint:   git log -p -- 'pkg/util/compat.go'
   ```
   - Reward = `hp + round(0.25*hp)` = `round(1.25*hp)` XP with math.Round
     half-away-from-zero (hp 50 → 63; the single formula shared with
     28/32/33).
   - Hint path is sanitized FIRST (31), then shell-quoted with a specified
     helper: wrap in single quotes with embedded `'` escaped as `'\''`
     (POSIX-safe). Quoting is display convenience, not a substitute for
     sanitization.
3. Target monster resolved from state by MonsterID; a dangling reference
   (missing monster — should not happen) renders
   `⚠ corrupted quest (target missing)` + stderr warning, exit stays 0.
   This is render-side degradation only — no engine change; the quest
   naturally resolves on a later scan via issue 28's cancellation rules.
4. Zero active quests + monsters alive: `No open bounties — the board refills
   on your next scan.`; zero because dungeon cleared: the celebratory line
   (same as status).
5. `--json`: `[{id, monster_id, issued_at, monster: {…report shape…},
   reward_xp}]` — schema golden.

## Acceptance Criteria

- [ ] Goldens: 3-quest board, 1-quest board, empty (both variants), dangling
      target degradation.
- [ ] Reward arithmetic golden (hp 50 → 63; hp 2 → 3 — the shared
      `hp + round(0.25·hp)` formula).
- [ ] Shell-quote helper: path `pkg/o'brien/x.go` renders as
      `'pkg/o'\''brien/x.go'` (golden).
- [ ] JSON schema golden; monster sub-object parity with report shape.
- [ ] Hostile path in hint line renders sanitized and quoted.

## Validation

```sh
go test -race ./internal/cli/ -run TestQuests
```

## Dependencies

- 10, 28, 30, 31.

## Non-goals

- Quest mutation (scan-only), acceptance flow (none in v1).

## Design References

- DESIGN §3.6, §8.2 (quests row).
