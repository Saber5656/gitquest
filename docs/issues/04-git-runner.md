# Title

Hardened read-only git subprocess runner (`internal/gitio.Runner`)

## Summary

Implement the single choke point through which every git invocation flows:
allowlisted subcommands, scrubbed environment, no shell, timeouts, capped
stderr, and a version preflight. This runner enforces product invariant I-1
(never write the target repository).

## Context

ADR-002 chose system-`git` subprocess over go-git. DESIGN §5.1/§10.6 define the
hardening rules. All later gitio issues (05–08) consume this runner.

## Scope

`internal/gitio/runner.go` (+ tests):

```go
type Runner struct { /* gitPath, repoDir(gitdir), timeout, logger */ }
func NewRunner(repoGitDir string, opts ...Option) (*Runner, error)
// Run executes an allowlisted read-only git command.
func (r *Runner) Run(ctx context.Context, args ...string) (stdout []byte, err error)
// Start streams stdout for long outputs (log, cat-file --batch).
func (r *Runner) Start(ctx context.Context, args ...string) (io.ReadCloser, WaitFunc, error)
func GitVersion(ctx context.Context) (semver string, err error)
```

## Detailed Requirements

1. Allowlist: first arg must be one of `version`, `rev-parse`, `rev-list`,
   `log`, `ls-tree`, `cat-file`, `blame`, `diff-tree`. Anything else returns
   `ErrForbiddenSubcommand` (and fails a dedicated unit test enumerating the list).
2. Every invocation is built as:
   `git --no-pager --no-optional-locks <subcommand> <args…>`; callers must
   precede pathspecs with `--`. The runner does NOT inspect or reject
   arguments after `--` — argv-array execution plus the `--` separator is the
   protection; legitimate tracked paths may begin with `-` (path VALIDITY
   rules — relative, no `..`, no NUL — are the parsers' job per DESIGN §10.2).
3. Child environment is constructed from scratch (never `os.Environ()`):
   `PATH`, `HOME`, `LC_ALL=C`, `GIT_TERMINAL_PROMPT=0`, `GIT_OPTIONAL_LOCKS=0`,
   `GIT_CONFIG_NOSYSTEM=0` (system config stays honored, stated explicitly),
   `GIT_DIR=<repoGitDir>` (absolute). `GIT_WORK_TREE`, `GIT_INDEX_FILE`,
   other `GIT_CONFIG_*` variables, and `GIT_ALTERNATE_OBJECT_DIRECTORIES`
   are NEVER present.
4. `git` binary resolved once via `exec.LookPath("git")` at `NewRunner`;
   resolved absolute path reused for all calls.
5. Default per-call timeout 120 s (Option to change); on timeout the process
   group is killed and the error says which command timed out (sanitized).
6. stderr captured into a 64 KiB-capped buffer; included in returned errors as
   an opaque field the CLI only prints under `-v` after sanitization (the
   runner itself must not print).
7. `GitVersion` parses `git version X.Y.Z…` and returns error `ErrGitTooOld`
   if < 2.30; `NewRunner` performs this probe once.
8. Zero third-party dependencies in this package (stdlib only).
9. No API for writing: the runner exposes no way to set `GIT_WORK_TREE`, no
   `-C` passthrough, no arbitrary env injection.

## Acceptance Criteria

- [ ] Unit test proves a non-allowlisted subcommand (`status`) is rejected
      without spawning a process (assert via fake exec hook or PATH shim).
- [ ] Unit test proves child env contains exactly the specified keys (spawn
      `git version` with an env-dumping PATH shim, or expose `buildEnv` for test).
- [ ] Timeout test: a stubbed slow git (shell script on PATH in a temp dir)
      is killed and reports `context deadline exceeded` with the command name.
- [ ] `GitVersion` accepts `git version 2.39.5 (Apple Git-154)` and rejects `2.29.0`.
- [ ] stderr larger than 64 KiB is truncated with a `…[truncated]` marker.
- [ ] `go test -race` passes; `gosec` reports no G204 finding (or a justified
      `#nosec` with comment explaining argv-array + allowlist).

## Validation

```sh
go test -race ./internal/gitio/... 
# Manual: in any repo, a debug main calling Run("rev-parse","HEAD") returns the SHA;
# `strace`/`fs_usage` optional spot-check: no lock files created in .git/.
```

## Dependencies

- 01.

## Non-goals

- Repo discovery/identity (05), stream parsers (06–08).
- No retry logic; callers decide.

## Design References

- DESIGN §5.1, §5.2, §10.6; ADR-002; invariant I-1.
