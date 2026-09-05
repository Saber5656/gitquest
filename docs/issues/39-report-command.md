# Title

`report` command: human debt report and the stable JSON schema

## Summary

Implement `gitquest report` — the practical deliverable: a human-readable
deletion-candidate report and, with `--json`, the frozen machine schema
(DESIGN §8.3) with `--fail-on-monsters` for CI gates.

## Context

This is where "playful + useful" cashes out (§1.3): the same data that renders
as monsters must read as an actionable tech-debt list. The JSON schema is a
public compatibility contract.

## Scope

`internal/cli/report.go`, `internal/cli/reportschema.go` (+ tests).

Flags: `--json`, `--fail-on-monsters`.

## Detailed Requirements

1. Read-only, no lock. Never-scanned: human mode prints the hint (exit 0);
   `--fail-on-monsters` without any scan → exit 1 with
   `no scan data — run 'gitquest scan' first` (a CI gate must not silently
   pass on missing data; deliberate difference from status).
2. Human mode layout: grouped by species, each living monster as an
   actionable line:
   ```
   ── Zombies (commented-out code) ─ 4 ──────────────
   src/legacy/api.js:120-154   34 lines   Grubmaw the Forgotten
     → delete the commented block, or: gitquest tame 'zombie:src/legacy/api.js:9f2c…'
   ```
   Ends with the summary line + `Tamed: N (excluded)` footer. Practical tone:
   paths first, names second (inverse of the game views — this is the
   "useful" face).
3. JSON document: exactly DESIGN §8.3 top-level fields
   (`schema_version: 1, generated_at, repo{id, root, head_sha},
   player{xp, level, xp_into_level, xp_to_next_level, lifetime{…}},
   boss, monsters[], quests[], achievements[] (unlocked only),
   last_scan{sha, at}, scan_health{warnings[], skipped_files, budget_hits[]}`).
   - `generated_at`: unix seconds at generation (the ONLY wall-clock field;
     everything else from state).
   - Monster objects: state shape (§7.2) + `name`, `depth`; absolute paths
     appear nowhere except `repo.root`.
   - Encode with `encoding/json` (control chars auto-escaped — the JSON
     injection defense; test asserts `` escaping of a hostile fixture).
4. Schema freeze mechanism: `reportschema.go` holds a JSON Schema (draft
   2020-12) document as a string constant; tests validate example outputs
   against it (use `santhosh-tekuri/jsonschema` as a TEST-ONLY dependency —
   it must not enter the binary; enforce via build-tag or test package +
   a `go list -deps` check in CI (46)).
   Field additions within v1 are allowed; removals/renames bump
   `schema_version` (documented in the constant's header comment).
5. `--fail-on-monsters`: living monsters ≥ 1 → exit 5 AFTER emitting the full
   report (CI logs show what failed the gate).
6. `scan_health` is copied verbatim from the state's `last_scan_health`
   block (DESIGN §7.2). Issue 32 owns writing that field; this command only
   reads it and never recomputes health.

## Acceptance Criteria

- [ ] Human golden (mono, 80 col): mixed-species fixture, cleared dungeon,
      all-tamed edge.
- [ ] JSON golden validates against the embedded schema; hostile strings
      appear `\u`-escaped; no absolute paths beyond repo.root (regex sweep in
      test).
- [ ] `monsters --json` ↔ `report --json` monster-object parity (shared test
      with 35).
- [ ] Exit codes: 0 (clean), 5 (monsters + flag), 1 (no scan + flag), 0
      (no scan, human mode, hint).
- [ ] Schema validator dependency proven absent from the release binary
      (`go list -deps ./cmd/gitquest` grep in test).
- [ ] `schema_version` bump policy documented in the schema constant comment.

## Validation

```sh
go test -race ./internal/cli/ -run TestReport
./gitquest report --json | jq .schema_version   # manual smoke: 1
```

## Dependencies

- 10, 30, 31, 32 (last_scan_health producer).

## Non-goals

- SARIF/badge outputs (v2); trend/history reporting; per-author data
  (privacy, §10.8).

## Design References

- DESIGN §8.3, §1.3, §10.3 (JSON escaping), §11 (exit 5), §7.2.
