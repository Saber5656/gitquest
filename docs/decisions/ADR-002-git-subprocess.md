# ADR-002: Git access via system `git` subprocess (read-only), not go-git

Status: Accepted

## Context

GitQuest reads commit history, trees, blobs, and line ages. Options:
pure-Go `go-git`, `libgit2` bindings (cgo), or shelling out to system `git`.

## Decision

Shell out to the system `git` binary through a single hardened runner
(`internal/gitio.Runner`) with:

- subcommand allowlist: `version`, `rev-parse`, `rev-list`, `log`, `ls-tree`,
  `cat-file`, `blame`, `diff-tree` (all read-only plumbing/porcelain),
- argv-array execution (no shell), `--` path separators, `--no-pager`,
- scrubbed child environment (never inherit `GIT_DIR`/`GIT_WORK_TREE`/`GIT_INDEX_FILE`;
  set `GIT_OPTIONAL_LOCKS=0`, `GIT_TERMINAL_PROMPT=0`, `LC_ALL=C`),
- per-call timeouts and size-capped stderr capture,
- minimum supported git: 2.30 (probed at startup; older ⇒ exit 3).

Content is read from blobs via `cat-file --batch`, never from worktree files.

## Consequences

- Requires `git` on PATH — acceptable: the product's audience is git users by
  definition; this is documented as a hard runtime dependency.
- Much faster history streaming on large repos than go-git; `blame` for free.
- The allowlist + env scrubbing make the read-only invariant (DESIGN §1.5 I-1)
  and hook-safety auditable in one file.
- Blob-based content reads eliminate worktree symlink traversal and TOCTOU
  (DESIGN §10.4) — a security simplification go-git would not give us by default.

## Alternatives considered

- go-git: no external dependency, but slow `log` on large repos, no blame,
  and worktree/file edge cases would push us to read the worktree directly.
- libgit2/cgo: breaks single-binary simplicity and cross-compilation.
