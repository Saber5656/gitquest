# Title

`tame` command: mark false positives as tamed (ignore), list, and undo

## Summary

Implement `gitquest tame <target>` (mark a living monster tamed),
`tame --list` (tamed registry), and `tame --undo <id>`, per DESIGN §8.2 and
the taming semantics of §3.5.

## Context

Taming is the product's false-positive escape hatch (§1.3) — it must be
frictionless and reversible. It mutates state, so it takes the profile lock.

## Scope

`internal/cli/tame.go` (+ tests).

Usage forms:
```
gitquest tame <monster-id>
gitquest tame <path>            # unique living monster on that path
gitquest tame <path>:<line>     # monster whose span covers that line
gitquest tame --reason "vendored on purpose" <target>
gitquest tame --list
gitquest tame --undo <monster-id>
```

## Detailed Requirements

1. Locking: exclusive profile lock for mutate forms; `--list` is read-only
   (no lock).
2. Target resolution order: exact monster id → exact path (all statuses
   considered but only `alive` is tameable) → path:line span containment.
   Ambiguity (≥ 2 living matches) → exit 2, stderr lists candidate ids +
   one-line descriptions (sanitized) — the user re-runs with an id.
   No match → exit 2 `no living monster matches <target>` (+ nearest-path
   suggestion if a tamed/slain monster matches: `did you mean --undo?` hint
   when a TAMED one matches).
3. Mutation: status → `tamed`; `Resolved = &Resolution{Status: tamed,
   At: now, Commit: ""}` — persisted JSON is
   `{"status":"tamed","at":<now>,"commit":""}` (`commit` present as an empty
   string, per issue 14's Resolution contract);
   `monster_tamed` event appended to the ring; optional `--reason` stored in
   the Monster field `tame_reason` (string ≤ 200 chars; already part of the
   DESIGN §7.2 schema and issue 10's structs). Quests targeting this monster:
   leave untouched — quest cancellation happens on next scan (28); print an
   FYI line if an active quest targets it (`the bounty on this monster will
   be withdrawn on your next scan`).
4. `--undo <id>`: only valid on `tamed` monsters → status back to `alive`
   (it will be re-verified next scan: if it no longer exists it will flee
   then — acceptable; document); event type: none in v1's closed set —
   do NOT invent one; append no event, print confirmation only (the closed
   event set of §3.10 is a compatibility contract).
5. Output: confirmation with the monster line
   (`🦴 Bonebag the Dusty is now tamed. It will no longer be reported.`);
   `--list`: table `NAME | SPECIES | ROOM | TAMED | REASON`.
6. XP: none for taming (DESIGN §3.5).
7. Timestamps: `time.Now().Unix()` via an injected clock (testability).

## Acceptance Criteria

- [ ] Resolution table tests: id, unique path, path:line, ambiguous → 2 with
      candidate list, no-match → 2, tamed-match → undo hint.
- [ ] Tame → state persisted (atomic), event in ring, quest FYI printed when
      applicable.
- [ ] Undo restores alive; undo on non-tamed → exit 2.
- [ ] `--reason` persisted and shown in `--list`; 201-char reason truncated
      with warning.
- [ ] Reconciliation integration (fixture): tamed monster still detected next
      scan stays tamed silently (26 req 4 verified end-to-end here).
- [ ] Lock held during mutation (concurrent tame → one wins, one exits 1).
- [ ] Injection safety (DESIGN §10.3): hostile monster name/path/evidence and
      a hostile `--reason` render sanitized in the confirmation line, the
      ambiguity candidate list (stderr), and `--list` table output.

## Validation

```sh
go test -race ./internal/cli/ -run TestTame
```

## Dependencies

- 10, 26, 30, 31.

## Non-goals

- Pattern-based ignores (config exclude covers that, 12/13); bulk taming;
  un-tame via re-scan.

## Design References

- DESIGN §3.5, §8.2 (tame row), §1.3, §9.2 (exclude as the bulk alternative).
