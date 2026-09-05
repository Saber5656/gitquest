# ADR-003: Non-interactive CLI core with a lightweight `explore` mode; full TUI game deferred

Status: Accepted (user-confirmed, 2026-07-10)

## Context

The concept ("repository as an RPG dungeon") could be built as a full roguelike
TUI, a plain reporting CLI, or a web app. v1 must be finishable, scriptable,
and still deliver the fantasy.

## Decision

- The product core is a set of non-interactive commands (`scan`, `status`,
  `map`, `monsters`, `quests`, `achievements`, `tame`, `report`, `reset`).
- One lightweight interactive command, `explore` (bubbletea), provides
  browse-only navigation (map/bestiary/quests/status) plus the tame action.
- Game logic lives in UI-free packages (`internal/game`, `internal/detect`);
  both CLI and TUI are thin renderers over the same state.
- A full TUI game loop (walking, battles, items) is explicitly v2.

## Consequences

- v1 completion risk is minimized; every feature is testable without a PTY.
- CI/scripting consumers get first-class JSON output (`report --json`).
- The core/renderer split means the v2 TUI game is additive, not a rewrite.
- The RPG feel in v1 depends on output quality (naming, events, styling) —
  DESIGN §3.9/§8.4 treat presentation as a real requirement, not polish.
