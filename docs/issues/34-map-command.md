# Title

`map` command: dungeon tree with monster placement

## Summary

Implement `gitquest map` — render the directory tree as dungeon floors with
monster counts, guardian markers, and the boss position, per DESIGN §8.2.

## Scope

`internal/cli/map.go` (+ golden tests).

Flags: `--depth N` (default 3), `--all` (include monster-free branches),
`--json` (tree as nested objects: `{name, depth, monsters: [ids],
guardian_id?, children: […]}`).

## Detailed Requirements

1. Read-only, no lock. Never-scanned → same hint/exit-0 behavior as status.
2. Tree construction from state monsters only — map does NOT read the git
   tree; it renders the last scan's knowledge (document this). Default view:
   branches containing ≥ 1 LIVING monster, plus the root-files floor
   (rendered as `Entrance Hall`, internal key `""` — DESIGN §3.2/issue 27)
   when it has living monsters. `--all` additionally includes branches whose
   only monsters are non-alive (slain/fled/tamed) — historical floors; it
   cannot show directories that never contained a monster (out of the state's
   knowledge; document).
3. Rendering (mono golden fixes exact glyphs):
   ```
   ⚔ The Dungeon of <repo>            23 monsters · Lv 11
   ├─ Entrance Hall         2 ▸ 🟢 zombie ×1  🟡 slime ×1
   ├─ src/                  14 ▸ 👑 Grubmaw the Forgotten (Ghost, Lv. 12)
   │  ├─ legacy/            9 ▸ guardian: Rattlejaw the Ancient
   │  └─ api/               5
   └─ pkg/                  7 ▸ guardian: Bonebag the Dusty
   ```
   - Node line: name, living-monster count in subtree, badges: boss 👑 on its
     floor path chain, `guardian:` on floor roots, species glyph summary at
     leaf-ish nodes (≤ depth cutoff).
   - Collapse: single-child chains merge (`a/b/c/` one line) — standard tree
     compression; monsters aggregate up to the displayed node.
4. `--depth` counts displayed levels below root; deeper content aggregates
   into the ancestor's count with a `+` marker (`src/deep/… +6`).
5. Deterministic ordering: dirs sorted lexicographically; `Entrance Hall`
   (internal key `""`) first.
6. Mono theme replaces glyphs with ASCII (`[B]` boss, `[G]` guardian,
   species letters `Z S G L M O` — fixed mapping documented in render (31)).
7. All names/paths sanitized (31).

## Acceptance Criteria

- [ ] Goldens (mono, 80 col): small fixture (2 floors), boss+guardian
      overlap fixture, deep nesting with `--depth 2` aggregation, `--all`
      contrast, cleared dungeon (`The halls are quiet.`).
- [ ] `--json` golden with nested structure; ids resolvable against
      `monsters --json`.
- [ ] Depth-collapse arithmetic: counts at every displayed node equal the sum
      of subtree monsters (property test on random trees).
- [ ] Hostile path fixture renders sanitized.

## Validation

```sh
go test -race ./internal/cli/ -run TestMap
```

## Dependencies

- 10, 27 (guardian data in state), 30, 31.

## Non-goals

- Interactive navigation (42), full filesystem listing, git tree reads.

## Design References

- DESIGN §3.2, §8.2 (map row), §3.7.
