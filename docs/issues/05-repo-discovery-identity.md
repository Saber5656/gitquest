# Title

Repository discovery, preflight checks, repo identity, and degraded modes

## Summary

Resolve the target repository from `--repo`/cwd, run the preflight battery
(version, bare, shallow, empty), and compute the stable repo identity used to
key the state profile.

## Context

DESIGN §5.2 lists the exact plumbing calls; §5.5 defines degraded modes; §7.1
defines identity (root commit SHA — lexicographically smallest when multiple —
plus slug). Every command starts here.

## Scope

`internal/gitio/repo.go` (+ tests):

```go
type Repo struct {
    Root      string // absolute worktree root
    GitDir    string // absolute .git dir
    HeadSHA   string
    RootSHA   string // identity root commit (lexicographically smallest root)
    Shallow   bool
    HeadTime  int64  // committer time of HEAD (RepoNow for detectors)
}
func Discover(ctx context.Context, startDir string) (*Repo, *Runner, error)
func (r *Repo) ID() string // first16hex(RootSHA) + "-" + Slug(basename(Root))
func Slug(name string) string
```

## Detailed Requirements

1. Discovery: `git rev-parse --show-toplevel` and `--absolute-git-dir` from
   `startDir` (a plain `--repo` path argument; note the runner has no `-C`, so
   discovery runs a bootstrap invocation with `Dir=startDir` — extend Runner
   with a discovery-only constructor if needed, keeping the allowlist).
2. Preflight order and error mapping (DESIGN §5.2 table order, §5.5, §11) —
   the git-version probe comes FIRST so no later plumbing call runs on an
   unsupported git:
   - git < 2.30 → `ErrGitTooOld` (exit 3; probe done once in `NewRunner`, 04);
   - not a repo → `ErrNotARepo` (exit 3) with hint `not a git repository (or any parent)`;
   - `rev-parse --is-bare-repository` true → `ErrBareRepo` (exit 3);
   - `rev-parse --verify HEAD` fails → `ErrEmptyRepo` (exit 3) message
     `This dungeon has no history yet — make your first commit.`;
   - `rev-parse --is-shallow-repository` true → `Repo.Shallow = true` (warning, not error).
3. Identity: `git rev-list --max-parents=0 HEAD`, read ALL root SHAs, and
   take the lexicographically smallest (DESIGN §7.1). This rule is fully
   deterministic regardless of traversal order or equal commit timestamps;
   note it in a code comment. (Single-root repos — the overwhelmingly common
   case — are unaffected.)
4. `Slug`: lowercase; keep `[a-z0-9-]`, map others to `-`; collapse repeats;
   trim leading/trailing `-`; cap 32 chars; empty → `repo`.
5. `HeadTime`: `git log -1 --format=%ct HEAD` (or fold into discovery batch).
6. All errors are typed and carry exit-code mapping via `cli.ExitCode` extension.

## Acceptance Criteria

- [ ] Table-driven tests: temp repos for normal / bare / empty / shallow
      (created with real `git` in `t.TempDir()`), plus non-repo dir → each
      yields the specified typed error or flags.
- [ ] Multi-root repo fixture (two orphan branches merged) → `RootSHA` equals
      the lexicographically smallest root SHA, stable across runs.
- [ ] `Slug("My Répo!! (2)")` == `my-r-po-2`; `Slug("")` == `repo`;
      33+ char names truncated to 32.
- [ ] `ID()` format matches `^[0-9a-f]{16}-[a-z0-9-]{1,32}$`.
- [ ] Discovery from a subdirectory of the repo finds the same Root.

## Validation

```sh
go test -race ./internal/gitio/ -run 'TestDiscover|TestSlug|TestIdentity'
```

## Dependencies

- 04.

## Non-goals

- Profile directory creation (09). History/tree reading (06/07).

## Design References

- DESIGN §5.2, §5.5, §7.1, §11.
