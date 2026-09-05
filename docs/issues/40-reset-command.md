# Title

`reset` command: soft reset and hard profile deletion

## Summary

Implement `gitquest reset` (soft: clear monsters/quests/events, keep
XP/achievements) and `reset --hard` (delete the entire profile directory),
with confirmation gates, per DESIGN §8.2.

## Scope

`internal/cli/reset.go` (+ tests). Flags: `--hard`, `--yes`.

## Detailed Requirements

1. Locking: exclusive profile lock for both modes (hard mode: acquire, then
   delete the profile dir including the lock file — release-after-delete is a
   no-op; document the ordering).
2. Confirmation: interactive TTY prompt
   (`This will delete <profile dir> — the dungeon forgets everything. [y/N]`
   for hard; softer copy for soft). Non-TTY without `--yes` → exit 2
   `refusing to reset without --yes in non-interactive mode`. `--yes` skips.
3. Soft reset: monsters, quests, events, quest_counter, last_scan_health →
   zeroed; player (xp/level/lifetime/authors), achievements, repo identity,
   AND `last_scan` → PRESERVED. `last_scan` is deliberately NOT cleared:
   the next scan runs as a normal INCREMENTAL scan (no first-scan XP replay,
   so double-awarding is structurally impossible), and because the previous
   monster set is empty, every current finding re-appears as a new monster
   (bulk `monster_appeared` is expected and correct here). The cache is left
   intact (still valid).
4. Hard reset: `os.RemoveAll(profileDir)` — path MUST be revalidated before
   removal: absolute, under DataRoot, matches the profile-dir pattern
   (`[0-9a-f]{16,40}-[a-z0-9-]+`); refuse otherwise (defense in depth against
   path bugs — §10.4).
5. Output: what was kept/lost, one line each; suggest `gitquest scan` after.
6. Deleting a non-existent profile (never scanned): hard → friendly no-op
   exit 0; soft → hint + exit 0.

## Acceptance Criteria

- [ ] Soft reset fixture: XP/achievements/last_scan survive; monsters/quests/
      events empty; next scan is incremental (asserted via state
      last_scan.sha unchanged pre-scan), re-detects monsters as new
      appearances, re-issues quests, and player XP grows only by the new
      commits' XP (exact arithmetic asserted — no historical replay).
- [ ] Hard reset removes the dir; a second hard reset no-ops politely.
- [ ] Path revalidation: forged Profile pointing outside DataRoot → refused
      with error (unit test on the guard function).
- [ ] Non-TTY without `--yes` → exit 2; with `--yes` proceeds.
- [ ] TTY prompt: `n` (and EOF) abort with exit 0 and no changes.

## Validation

```sh
go test -race ./internal/cli/ -run TestReset
```

## Dependencies

- 09, 10, 30, 31 (+ behavioral contract with 32's XP guard).

## Non-goals

- Resetting user config; multi-repo cleanup (`gitquest gc` is a v2 idea).

## Design References

- DESIGN §8.2 (reset row), §7.3 (recovery messaging), §10.4 (path confinement).
