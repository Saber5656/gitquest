# ADR-004: State in the user's home data dir; target repo strictly read-only; fully offline

Status: Accepted (user-confirmed, 2026-07-10)

## Context

Player progression (XP, monsters, tames) needs persistence. Options: inside the
repository (`.gitquest/`, committable and team-shareable), user home directory,
or stateless recomputation. Independently, the product could phone home
(telemetry/update checks) or stay offline.

## Decision

1. All state lives under `${XDG_DATA_HOME:-~/.local/share}/gitquest/profiles/<repo-id>/`
   (override: `GITQUEST_DATA_DIR`). Repo identity = root-commit SHA
   (lexicographically smallest when multiple roots exist) + sanitized
   basename slug. The target repository is never written — no
   files, refs, config, or locks (enforced by ADR-002's runner design).
2. GitQuest performs no network I/O of any kind. No telemetry, no update
   checks. Absence is verified by a CI dependency/import check.
3. Stateless recomputation was rejected: slay/fled event detection and tame
   persistence require prior-scan memory; caches make incremental scans fast.

## Consequences

- Safe to run on any repository, including other people's checkouts; nothing
  to gitignore; no merge conflicts from game files.
- Team-shared progression is impossible in v1 (accepted; per-author-ready
  data model keeps v2 options open — DESIGN §7.2).
- Repo-local `.gitquest.toml` remains possible for shared *configuration*
  (read-only, untrusted input — DESIGN §9.2).
- Deleting the repo leaves an orphaned profile; `reset --hard` and a
  documented data-dir layout mitigate.
