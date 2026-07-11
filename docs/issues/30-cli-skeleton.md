# Title

CLI skeleton: root command, global flags, exit-code mapping, command context wiring

## Summary

Extend the issue-01 stub into the full command scaffold: all global flags,
the shared per-command bootstrap (discover repo → open profile → load state
→ resolve config), typed-error → exit-code mapping, and stdout/stderr
discipline, per DESIGN §8.1/§11.

## Scope

`internal/cli/root.go`, `context.go` (+ tests). Registers stub subcommands
(`scan` etc. are separate issues; each stub prints "not implemented" and
exits 1 until its issue lands — stubs let this issue finalize wiring).

```go
type App struct { // assembled per invocation, passed to every command
    Repo    *gitio.Repo
    Runner  *gitio.Runner
    Store   *state.Store
    Cfg     config.Config
    CfgWarn []config.Warning
    Out     io.Writer // stdout
    Err     io.Writer // stderr
    JSON    bool
    Render  *render.Renderer // nil until issue 31 lands; stubs guard
}
func bootstrap(cmd *cobra.Command) (*App, error) // shared pre-run
```

## Detailed Requirements

1. Global flags exactly (DESIGN §8.1): `--repo PATH` (default "." discovery),
   `--json` (registered globally, honored by commands that document it;
   others reject with usage error exit 2 `--json is not supported by this
   command`), `--no-color`, `--theme auto|dark|light|mono` (enum-validated),
   `--config PATH` (user config override), `-v/--verbose`.
2. `NO_COLOR` env ⇒ same as `--no-color` (precedence: `--theme mono` >
   `--no-color`/`NO_COLOR` > `--theme` > auto-detect; final resolution
   implemented in 31, flag plumbing here).
3. Exit-code mapping table (extends issue 01's `ExitCode`): typed errors from
   gitio/state map to 3/4; `ErrLockBusy` → 1; cobra flag errors → 2;
   `--fail-on-monsters` sentinel `ErrMonstersPresent` → 5. Unit-tested with
   every typed error.
4. Bootstrap order & failure behavior (DESIGN §4.2 steps 1–3): repo discovery
   failures exit BEFORE profile creation (no empty profiles for non-repos).
   Config warnings are buffered in App and flushed to stderr once a Renderer
   exists (31). Until 31 lands, the interim flush passes warning text through
   a minimal control-byte stripper (strip 0x00–0x1F except \n/\t, and 0x7F) —
   warnings can quote repo-derived `.gitquest.toml` content and must never
   emit raw control bytes (DESIGN §10.3), even in this stub phase. The
   stripper is replaced by `render.Sanitize` in 31.
5. Read-only commands (`status`, `monsters`, …) must NOT take the profile
   lock; only `scan`, `tame`, `reset` do (each command issue states its
   locking; bootstrap exposes `Store.AcquireLock` but never calls it).
6. stdout/stderr rule (DESIGN §8.1): human output → stdout; warnings/errors →
   stderr; `--json` mode: ONLY the JSON document on stdout (warnings still
   stderr). Enforced by a helper `App.Warnf` and tested.
7. `version` subcommand: `gitquest <version> (<commit>, <date>)` — ldflags
   vars from issue 01; also `--version` root flag aliasing it.
8. Command registry pattern: each command file exports
   `func newScanCmd(app func() *App) *cobra.Command`; root assembles. Stubs
   for all 11 commands registered NOW so `gitquest help` shows the full tree
   (helps docs; each prints not-implemented until its issue).

## Acceptance Criteria

- [ ] `gitquest --help` lists all 11 commands with one-line descriptions
      matching DESIGN §8.2's table wording.
- [ ] Exit-code table test: forged typed errors produce 1/2/3/4/5 as mapped.
- [ ] `gitquest status --json` on a stub → exit 1 (not implemented; `status`
      is a `--json`-capable command per DESIGN §8.1, so the flag passes flag
      validation and reaches the stub). `gitquest tame --json` → exit 2
      (tame is not `--json`-capable). `gitquest badflag` → 2. Non-repo dir →
      3 BEFORE any profile dir is created (assert data dir untouched).
- [ ] `NO_COLOR=1` reaches the App's theme resolution inputs (unit test on
      the flag plumbing).
- [ ] `--json` on a non-JSON command → exit 2 with the specified message.
- [ ] Golden `--help` output committed (guards accidental UX drift).

## Validation

```sh
go test -race ./internal/cli/ -run 'TestRoot|TestExitCodes|TestBootstrap'
go build -o gitquest ./cmd/gitquest && ./gitquest --help
```

## Dependencies

- 01, 04, 05, 09, 10, 12 (bootstrap pieces). 31 lands right after (Renderer).

## Non-goals

- Any real command behavior (32–40); styling (31); shell completions (v2).

## Design References

- DESIGN §8.1, §8.2 (table), §11, §4.2.
