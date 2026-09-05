# Title

`scan` command: full pipeline wiring and event-feed output

## Summary

Implement `gitquest scan` — the only command that advances game state. Wires
the entire DESIGN §4.2 sequence: history → XP → tree → detection →
reconciliation → quests/boss/achievements → persist → render events.

## Context

This is the integration point of waves 1–3. Everything it composes already
exists and is unit-tested; this issue is about correct orchestration,
transactional persistence, and the scan UX.

## Scope

`internal/cli/scan.go` (+ integration-style tests using fixture repos).

Flags: `--fail-on-monsters`, `--quiet` (suppresses event feed, keeps summary
line), plus globals. Supports `--json` per DESIGN §8.1/§8.3: emits the §8.3
report document plus one additional array `events_this_scan` (same schema
family as 39; reuse its encoder).

## Detailed Requirements

1. Locking: acquire the profile lock for the WHOLE run (bootstrap → persist);
   busy ⇒ exit 1 (message per issue 10).
2. Pipeline order exactly DESIGN §4.2 (steps 1–11) with these wiring details:
   - First scan (`state.LastScan == nil`): full history walk; build cache
     from stream (`Cache.ApplyCommit` per commit); XP via
     `game.FoldCommits` of ALL commits. Event policy (DESIGN §3.4/§3.5/§3.7):
     suppress per-commit `xp_gained`, individual `level_up` (both folded
     into the single `first_scan_completed` event carrying commits/XP/level),
     and bulk `monster_appeared`; still emit `quest_issued`,
     `achievement_unlocked` (incl. Night Watch), and the single
     `boss_appeared`.
   - Incremental: `Reachable(last.sha)` false ⇒ rewrite recovery (DESIGN
     §5.4): warning, rebuild cache full, XP for commits not previously
     counted — v1 approximation: count first-parent commits reachable from
     HEAD; if `count(HEAD) > player.lifetime.commits_scanned`, award XP for
     the newest `count - commits_scanned` commits (walk with
     `--max-count`); never subtract. Document this approximation in godoc
     and in `report --json` scan_health when triggered.
   - TouchedPaths built from the incremental numstat stream (path → last
     commit SHA); renames from `RenameMap(last.sha, HEAD)`.
   - CommitFact adapter: binary numstat entries contribute 0/0 (15);
     AuthorKey = `state.HashAuthor(AuthorMail)`.
   - RepoAlive = any commit `When ≥ HeadTime-90d` (checked over the cache’s
     newest entries + this scan's commits; on first scan derived during the
     walk).
   - Detection input assembled per issue 19; species disabled by config
     filtered.
   - Reconcile (26) → QuestUpdate (28) → Boss events (27; called on every
     scan including the first) → XP application in one deterministic,
     documented order: commit XP, then per resolved monster its slay value
     (`hp`, or `hp + round(0.25·hp)` = `round(1.25·hp)` when the slay
     completes an active quest — single formula shared with 28/33/36), then
     level computation & `LevelUps` events (16).
   - Achievements last (29), with facts assembled from the above.
   - Persist state + cache atomically (10/11) ONLY if every prior step
     succeeded (DESIGN §4.2: failures abort without persisting). Persisted
     state includes `last_scan_health` (warnings, skipped_files, budget_hits —
     DESIGN §7.2); `report` (39) reads it verbatim.
3. Output (non-JSON): event feed via `Renderer.EventLine` in this order:
   first_scan_completed | (level_ups) | slain | fled | appeared (capped at 20
   lines with `… and N more appeared` overflow) | quest events | boss events |
   achievements; then the one-line summary (DESIGN §8.2):
   `Lv 11 · 15,230 XP · 23 monsters (1 boss) · 3 quests`. `--quiet`: summary
   only. Scan warnings (budget hits, shallow, skipped counts) go to stderr.
4. `--fail-on-monsters`: after persist + output, if living monsters ≥ 1 →
   exit 5 (`ErrMonstersPresent` through the 30 mapping).
5. Duration: print `scanned N files in X.Xs` as part of stderr info line when
   `-v`.
6. Determinism: given a fixture repo and fixed dates, two runs produce
   byte-identical non-JSON output (mono theme) — golden test.

## Acceptance Criteria

- [ ] E2E happy path on a scripted fixture repo: first scan output golden
      (mono, 80 cols), state.json golden (volatile fields normalized).
- [ ] Incremental scan after a "slay commit" (delete a zombie block):
      `monster_slain` + slay XP + quest completion (×1.25 total verified in
      state XP arithmetic) + possible level_up — all in golden output.
- [ ] Threshold-change flee: raise zombie_min_lines via repo config commit →
      `monster_fled`, zero XP delta from it.
- [ ] Rewrite recovery: amend the fixture's history → warning + no XP loss
      (lifetime XP unchanged or grown; assert monotonic).
- [ ] `--fail-on-monsters` exits 5 with monsters, 0 when dungeon cleared.
- [ ] Failure injection: make persist fail (read-only profile dir) → exit 1,
      previous state intact, lock released.
- [ ] `--json` output validates against the schema fixture of 39 (shared
      validator) and includes `events_this_scan`.
- [ ] Second concurrent scan (spawned process on same profile) → exit 1,
      lock message.
- [ ] Injection safety (DESIGN §10.3): a hostile fixture (ANSI in file path,
      TODO text, and conflict-marker branch label) produces event feed,
      warnings, and `-v` git-stderr output with zero raw 0x1b/C0 bytes, and
      `--json` output with those strings `\u`-escaped.

## Validation

```sh
go test -race ./internal/cli/ -run TestScan
# plus the shared fixture harness once 44 lands (44 re-uses these scenarios)
```

## Dependencies

- 06, 07, 10, 11, 15, 16, 19, 26, 27, 28, 29, 30, 31.

## Non-goals

- Report formatting beyond the feed/summary (39), map view (34), TUI (41).

## Design References

- DESIGN §4.2, §3.4 (XP order), §3.5, §5.3–§5.5, §8.2, §8.3, §11.
