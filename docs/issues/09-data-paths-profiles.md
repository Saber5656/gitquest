# Title

Data directory resolution and per-repository profile directories

## Summary

Resolve the GitQuest data/config roots (env override → XDG → home fallback),
create per-repo profile directories keyed by repo identity, and enforce
private permissions.

## Context

DESIGN §7.1 defines locations, the `repo-id` directory naming, collision
fallback, and permissions. ADR-004 fixes home-dir storage. Everything the
store (10) and cache (11) write lives here.

## Scope

`internal/state/paths.go` (+ tests):

```go
type Paths struct{ DataRoot, ConfigRoot string }
func ResolvePaths(getenv func(string) string) (Paths, error)

type Profile struct{ Dir string /* absolute */ }
// OpenProfile ensures the profile dir exists (0700) for the given repo.
func OpenProfile(p Paths, repoID string, rootSHA string) (Profile, error)
func (p Profile) StatePath() string  // state.json
func (p Profile) BakPath() string    // state.json.bak
func (p Profile) CachePath() string  // cache.json
func (p Profile) LockPath() string   // lock
```

## Detailed Requirements

1. Resolution order (each independently):
   - data: `GITQUEST_DATA_DIR` → `$XDG_DATA_HOME/gitquest` → `~/.local/share/gitquest`
   - config: `GITQUEST_CONFIG_DIR` → `$XDG_CONFIG_HOME/gitquest` → `~/.config/gitquest`
   Relative env values are an error (exit 1, message names the variable).
   Same rules on all OSes including macOS/Windows (DESIGN §7.1: predictable
   CLI-convention paths; document in code comment).
2. `OpenProfile`:
   - input validation FIRST (§10.4 — this function joins its inputs into
     filesystem paths): `repoID` must match `^[0-9a-f]{16}-[a-z0-9-]{1,32}$`
     and `rootSHA` must match `^[0-9a-f]{40}$`; anything else (path
     separators, `..`, uppercase, wrong length) → typed error, no directory
     touched;
   - primary dir name = `repoID` (already `first16hex-slug` from issue 05);
   - collision rule: if the dir exists AND contains a `state.json` whose
     `repo.root_sha` ≠ `rootSHA` (peek: decode only that field, tolerate
     corrupt file as "unknown" → treat as collision), fall back to
     `<full40hex>-<slug>`;
   - `MkdirAll` with 0700 for `DataRoot`, `profiles/`, and the profile dir;
     tighten pre-existing looser modes via `os.Chmod` (best effort on
     platforms without POSIX perms).
3. All files later created in the profile must be 0600 — expose a helper
   `CreatePrivate(path string) (*os.File, error)` used by 10/11.
4. No file contents are read/written here beyond the collision peek.
5. Pure function style: `ResolvePaths` takes `getenv` for testability; no
   global state.

## Acceptance Criteria

- [ ] Table tests for each root (data AND config, separately): (a) env
      override wins over XDG and home; (b) XDG set + env unset → XDG path;
      (c) both unset → home fallback (`~` via `os.UserHomeDir`); (d) relative
      env value → typed error naming the variable. 8 cases total.
- [ ] `OpenProfile` input validation: `../evil`, `a/b`, `ABCDEF…` (uppercase),
      15-hex prefix, 41-hex rootSHA → typed error, no directory created.
- [ ] Profile creation: fresh dir is 0700; files via `CreatePrivate` are 0600
      (skip perm asserts on Windows with `runtime.GOOS` guard).
- [ ] Collision test: same 16-hex prefix + slug, different rootSHA in existing
      state.json → full-hash dir returned; corrupt existing state.json also
      routes to full-hash dir.
- [ ] Idempotent: calling `OpenProfile` twice returns the same dir.

## Validation

```sh
go test -race ./internal/state/ -run 'TestResolvePaths|TestOpenProfile'
```

## Dependencies

- 01, 05 (repoID format).

## Non-goals

- State serialization/locking (10), cache content (11), reset semantics (40).

## Design References

- DESIGN §7.1, §10.4 (path confinement), ADR-004.
