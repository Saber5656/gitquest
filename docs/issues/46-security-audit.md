# Title

Security audit: cross-cutting injection/DoS/isolation test suite and checklist execution

## Summary

Execute the DESIGN §10 security model as a dedicated verification pass: an
adversarial test suite (hostile repos, injection fixtures, resource bombs),
static guards (network imports, dependency policy), and a written audit
checklist with evidence, per DESIGN §10.1–§10.8.

## Context

Individual issues each carry local security acceptance criteria; this issue
verifies the WHOLE posture end-to-end and produces the audit record that the
v1.0.0 release (47) requires. Findings become fix issues.

## Scope

`internal/e2e/security_test.go` (extends 44's harness), `scripts/depcheck.sh`,
`docs/security-audit-v1.md` (the completed checklist with evidence links).

## Detailed Requirements

1. Hostile-repo corpus (fixtures built via 44's builder):
   - filenames with: ANSI escapes, OSC title-set, bidi overrides (RTLO),
     zero-width joiners, 255-byte names, deep nesting (50 levels),
     10k-file directory;
   - blob contents (real fixtures kept small): NUL-laced text, a single line
     of several MB (with a lowered test-only per-line/size limit proving the
     cap), malformed UTF-8. The 2 GiB declared-size guard is verified at
     UNIT level against a fake `cat-file --batch` stream that lies about
     size (07's reader), NOT with a real 2 GiB object — CI-realistic;
   - `.gitquest.toml`: 64 KiB+1 size, quadratic-blowup TOML (nested arrays),
     budget-raise attempts, absolute/`..` globs;
   - commit metadata: ANSI in author name, 100 KB commit subject,
     conflict-marker labels with escapes.
   Assertions: every command's stdout/stderr byte-swept for ESC (0x1b), raw
   C1 (0x80–0x9F), and C0 EXCEPT the layout-legitimate `\n`, `\t`, `\r`
   (DESIGN §8.4/§10.3); exit codes stay in the documented set; peak RSS
   bounded (child-process measurement from 45); no file created outside
   GITQUEST_DATA_DIR (fs-diff the repo worktree + a canary sibling dir
   before/after).
2. Read-only invariant proof: snapshot the target repo dir (recursive
   mtime+size+hash manifest) before/after every scenario in 44+this suite →
   byte-identical (this is I-1's executable proof; `.git` included —
   `--no-optional-locks` verified here).
3. Offline invariant proof: run the full e2e suite under a network-denying
   sandbox — implementation options (pick one, document): unshare -n (linux
   CI), or a `GODEBUG`-independent LD_PRELOAD-free approach: assert via
   `go list -deps` that no package under `cmd/ internal/` imports `net`,
   `net/http`, `net/rpc`, `os/exec`-spawned curl-likes (subprocess allowlist
   already constrains to git); PLUS the CI netguard grep (02). Both static
   checks land in `scripts/depcheck.sh` (also verifies the test-only
   jsonschema dep stays out of the binary — 39).
4. Checklist document `docs/security-audit-v1.md`: one row per DESIGN §10
   subsection: control, verification method, evidence (test name/CI run
   link), residual risk. "Out of scope" rows (§10.1) restated for the record.
5. Dependency policy check: `go mod graph` reviewed; direct deps must equal
   the DESIGN §10.7 target set (cobra, bubbletea, bubbles, lipgloss,
   doublestar, BurntSushi/toml, x/term + test-only jsonschema); any extra →
   PR discussion + DESIGN update or removal.
6. Fuzz corpus consolidation: ensure 07 (blob decode), 13 (repo config),
   31 (sanitizer) fuzz targets run in a scheduled weekly CI job
   (`fuzz.yml`, 10 min each, SHA-pinned).
7. Finding workflow (keeps the audit reviewable, no silent fixes): every
   finding is recorded in `docs/security-audit-v1.md`; findings requiring
   code changes become new draft files numbered `docs/issues/49-…` onward,
   `docs/ISSUE_PLAN.md` is updated to list them (as an audit-findings wave),
   and GitHub issues are then created from those drafts — same
   docs-first flow as the original 48.

## Acceptance Criteria

- [ ] Hostile-repo suite green with all byte-sweeps and fs-diff assertions.
- [ ] Read-only manifest proof passes on every e2e scenario (wired into 44's
      harness as a global fixture hook).
- [ ] `scripts/depcheck.sh` in CI: netguard + dep-policy + binary-purity all
      pass; deliberately adding `net/http` fails it (scratch-commit proof).
- [ ] `docs/security-audit-v1.md` complete — every §10 subsection has
      evidence or a filed follow-up issue.
- [ ] Weekly fuzz workflow exists and passed at least once (manual dispatch).

## Validation

```sh
make test-e2e && scripts/depcheck.sh
gh workflow run fuzz.yml && gh run watch
```

## Dependencies

- 44 (harness), 45 (RSS measurement), 02 (CI), 04/07/13/19/31/39 (controls
  under audit).

## Non-goals

- Fixing found bugs (separate issues); artifact signing (v2); formal
  pentest/third-party audit.

## Design References

- DESIGN §10 (entire), I-1, I-2, §13.
