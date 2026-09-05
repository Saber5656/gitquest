# Title

E2E fixture repository builder and integration scenario suite

## Summary

Build the deterministic fixture-repo generator and the end-to-end scenario
suite that drives the real `gitquest` binary through the full product
lifecycle, asserting on golden outputs and `report --json`, per DESIGN §13.

## Context

Unit tests lock formulas; this suite locks BEHAVIOR: the scenarios in
ISSUE_PLAN §6.3 are the product's executable specification. It also becomes
the regression bed for the security suite (46) and benchmarks (45).

## Scope

`internal/e2e/` (test-only package, build tag `e2e`), `Makefile` target
`test-e2e`, CI job addition.

```go
type RepoBuilder struct{ /* temp dir + git runner with env-pinned dates */ }
func NewRepoBuilder(t *testing.T) *RepoBuilder
func (b *RepoBuilder) Commit(msg string, files map[string]string, at time.Time, author string)
func (b *RepoBuilder) Move(old, new string, at time.Time)
func (b *RepoBuilder) Delete(path string, at time.Time)
func (b *RepoBuilder) Amend(at time.Time)               // history rewrite
func (b *RepoBuilder) ShallowClone(t *testing.T) string  // depth-1 clone dir
func RunGitquest(t *testing.T, repo string, env map[string]string, args ...string) (stdout, stderr string, exit int)
```

## Detailed Requirements

1. Determinism: every git op sets `GIT_AUTHOR_DATE`/`GIT_COMMITTER_DATE`
   (from the `at` param), fixed author/committer identities, `git -c
   commit.gpgsign=false`; builder asserts resulting SHAs are stable across
   runs on the same platform+git-version (record SHAs in goldens ONLY where
   necessary; prefer normalizing volatile fields).
2. `RunGitquest` executes the COMPILED binary (`go build` once per test run
   into t.TempDir via TestMain) with isolated `GITQUEST_DATA_DIR` and
   `GITQUEST_CONFIG_DIR` per test, `NO_COLOR=1`, `TZ=UTC`, 80-col
   `COLUMNS` hint.
3. Scenario suite (each = one test function, goldens under
   `internal/e2e/golden/`):
   S1 first scan (mixed monster fixture: ≥ 1 of each species);
   S2 incremental slay (+quest completion ×1.25 math assert via state diff);
   S3 fled via repo-config threshold commit;
   S4 tame → rescan silence → `tame --undo` → rescan re-alive;
   S5 rename migration (git mv, id rewritten, aka recorded);
   S6 history rewrite recovery (amend) — XP monotonic;
   S7 shallow clone — warning banner, no crash;
   S8 empty repo / non-repo / bare repo → exits 3 with messages;
   S9 dungeon cleared — achievement + celebratory status;
   S10 `--fail-on-monsters` exit-5 gate;
   S11 concurrent scan lock (spawn two, one exits 1);
   S12 hostile fixture: ANSI filename + evil TODO + conflict labels →
   byte-scan of ALL outputs for forbidden bytes: ESC (0x1b), C1 (0x80–0x9F
   as raw bytes), and C0 EXCEPT `\n`, `\t`, `\r` (which legitimate output
   uses — DESIGN §8.4/§10.3). Also feeds 46.
4. Golden discipline: `-update` flag regenerates; goldens normalized
   (volatile: absolute paths → `<ROOT>`, SHAs → `<SHA:n>` indexed, unix
   times → `<T+offset>`); normalizer is part of this issue and reused by 45/46.
5. JSON assertions: `report --json` parsed and schema-validated (39's
   validator) in every scenario that scans.
6. Runtime budget: full suite ≤ 90 s locally (parallel t.Parallel where
   fixtures independent).
7. CI: new job `e2e` (ubuntu + macos) runs `make test-e2e`.

## Acceptance Criteria

- [ ] All 12 scenarios green on ubuntu-latest and macos-latest.
- [ ] Golden regeneration is deterministic (two consecutive `-update` runs
      produce zero diff).
- [ ] S12's byte-sweep finds no ESC/C1/disallowed-C0 bytes in any
      stdout/stderr capture (allowed: `\n`, `\t`, `\r`).
- [ ] Builder API documented with an example test. (Runtime note for the PR
      description, not a gate: full suite target ≤ 90 s; the Go test timeout
      is the enforcement mechanism.)

## Validation

```sh
make test-e2e
```

## Dependencies

- 32 (scan), 33, 35, 38, 39 (commands exercised); 05 (error modes);
  grows as 34/36/37/40 land (add scenario steps opportunistically).

## Non-goals

- Performance measurement (45); TUI E2E (model tests in 41–43 cover it);
  Windows CI (best effort, excluded from the matrix).

## Design References

- DESIGN §13, ISSUE_PLAN §6.3, §5.4–§5.5, §10.3.
