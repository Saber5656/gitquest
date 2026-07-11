# Title

`monsters` command: bestiary table with filtering and sorting

## Summary

Implement `gitquest monsters` — the tabular bestiary over the state's monster
set with status/species filters and sorting, per DESIGN §8.2.

## Scope

`internal/cli/monsters.go` (+ golden tests).

Flags: `--species S` (validated enum, repeatable), `--status
alive|slain|fled|tamed|all` (default `alive`), `--sort hp|age|path` (default
`hp`, descending for hp/age, ascending for path), `--json`.

## Detailed Requirements

1. Read-only, no lock; never-scanned hint as in status.
2. Columns (Renderer.Table, PATH middle-truncated first at width 80):
   `NAME | SPECIES | LV | HP | ROOM | AGE | STATUS`
   - NAME: `Grubmaw the Forgotten` (+ ` 👑` if boss).
   - ROOM: `path` or `path:start-end` when span present.
   - AGE: since FirstSeen (`45d`, `1.2y` — render helpers).
   - STATUS styled: alive=species color, slain=strikethrough/muted,
     tamed=muted+`(tamed)`, fled=muted.
3. Footer: `23 alive · 17 slain · 2 tamed · 1 fled — 43 total` (always the
   full-state counts, independent of filter, so users see the whole picture).
4. `--json`: array of monster objects mirroring the report schema (39)
   monster shape exactly (single source: a shared `render/jsonshape` helper
   or the state struct's tags — implementation may reuse state structs; the
   CONTRACT is: byte-identical monster objects between `monsters --json` and
   `report --json`).
5. Sorting stability: ties broken by path then id (deterministic goldens).
6. Empty result under a filter: `No <species> monsters with status <s>.` +
   exit 0.
7. Row cap 200 with `… and N more (narrow with --species/--status)` overflow
   footer (protects terminals from 10k-monster repos).

## Acceptance Criteria

- [ ] Goldens: default view, `--status all`, `--species zombie --sort path`,
      empty-filter message, overflow at 201 monsters (fixture-generated).
- [ ] JSON parity test: same monster serialized via `monsters --json` and
      `report --json` is byte-identical.
- [ ] Boss marker appears on the boss row only.
- [ ] Invalid `--species dragon` → exit 2 usage error listing valid species.
- [ ] Hostile fixture: name/path/status strings containing ANSI/control
      bytes render sanitized in the table (evidence is not a table column);
      the same monster's `evidence` and other repo-derived fields appear
      `\u`-escaped in `--json` output (encoding/json contract, DESIGN §10.3).

## Validation

```sh
go test -race ./internal/cli/ -run TestMonsters
```

## Dependencies

- 10, 26 (status semantics), 30, 31.

## Non-goals

- Taming (38), detail view (42), pagination beyond the row cap.

## Design References

- DESIGN §8.2 (monsters row), §3.3, §8.3.
