# ADR-001: Implementation language and stack — Go + cobra + lipgloss/bubbletea

Status: Accepted (user-confirmed, 2026-07-10)

## Context

GitQuest is a CLI/terminal product for developers. Candidates: Go, TypeScript/Node, Rust.
Key forces: single-binary distribution (Homebrew/direct download), rich terminal
rendering for the RPG presentation, git-data processing performance, development
velocity, and the repository owner's ability to review the code.

## Decision

Go (latest stable, minimum 1.22), with:

- `spf13/cobra` — command tree, flags, help.
- `charmbracelet/lipgloss` — styled non-interactive output.
- `charmbracelet/bubbletea` (+`bubbles`) — the `explore` interactive mode.
- `BurntSushi/toml` — config parsing; `bmatcuk/doublestar` — glob matching.

## Consequences

- Single static binary (`CGO_ENABLED=0`), trivial installation, fast startup.
- Charm stack gives the "game feel" without a full TUI engine commitment.
- No AST tooling dependency pressure: v1 detection is heuristic by design (ADR-005),
  so Go's weaker multi-language AST ecosystem (vs Node) is not a constraint.
- Windows builds are nearly free but remain best-effort (not release-gating).

## Alternatives considered

- TypeScript/Node: fastest iteration, best AST ecosystem, but Node-runtime
  dependency and slower startup conflict with the zero-friction CLI goal.
- Rust: best performance and ratatui, but highest implementation cost for a
  product whose bottleneck is `git` subprocess I/O, not CPU.
