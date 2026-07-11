# Title

Release pipeline: goreleaser, checksums, version injection, tag workflow

## Summary

Implement the release machinery: goreleaser configuration for darwin/linux
artifacts plus a separate non-blocking Windows zip step, SHA256SUMS,
version/commit/date ldflags injection, a tag-triggered GitHub Actions release
workflow, and the release runbook, per DESIGN §14 and §10.7.

## Scope

`.goreleaser.yaml`, `.github/workflows/release.yml`, `docs/releasing.md`.

## Detailed Requirements

1. goreleaser:
   - builds: `CGO_ENABLED=0`, `-trimpath`, ldflags `-s -w -X
     github.com/Saber5656/gitquest/internal/cli.version={{.Version}}
     …commit={{.ShortCommit}} …date={{.CommitDate}}` (reproducible: commit
     date, not build date);
   - goreleaser targets: darwin/{amd64,arm64} and linux/{amd64,arm64}
     (tar.gz) ONLY. Windows/amd64 is deliberately NOT in the goreleaser
     matrix (a failing target there would block the whole release, violating
     DESIGN §2.3's "not release-blocking"); instead a separate
     `continue-on-error: true` workflow step cross-builds the windows zip
     and uploads it to the same draft release via `gh release upload` —
     its failure never gates the release;
   - archives include LICENSE + README; binary named `gitquest`;
   - `checksum: sha256` → `SHA256SUMS.txt`;
   - changelog: conventional grouping by commit prefix (feat/fix/docs/other);
   - snapshot config for CI dry-runs.
2. `release.yml`: trigger `push: tags: v*`; jobs: (a) full CI reuse (call
   ci.yml via `workflow_call`), (b) goreleaser release with
   `permissions: contents: write` ONLY on that job; actions SHA-pinned;
   no other secrets (GITHUB_TOKEN only).
3. Snapshot job on main merges (extends ci.yml): `goreleaser release
   --snapshot --clean` artifact-uploaded for 7 days (release rehearsal,
   ISSUE_PLAN §6.6).
4. Version stub from issue 01 must now produce release values:
   `gitquest v0.3.0 (abc1234, 2026-08-01)` — assert format in an e2e case.
5. `docs/releasing.md` runbook: preflight checklist (CI green, ISSUE_PLAN
   wave status, security audit for v1.0.0), tag command, verification steps
   (`sha256sum -c`, run binary on the three platforms), yank procedure
   (delete release + tag, publish fixed patch), and the repo-admin settings
   checklist that CANNOT be automated per repo rules (branch protection
   already exists; enable secret scanning + push protection + CodeQL — steps
   for the human owner, agent must not execute).
6. Homebrew tap: goreleaser `brews:` section PREPARED but commented out with
   a note (publishing to `Saber5656/homebrew-tap` requires a token and the
   tap repo — a human step per user rules; the config block ships inert).
7. No network calls at build beyond GitHub-provided actions/goreleaser
   (module proxy is CI-level, unaffected).

## Acceptance Criteria

- [ ] `goreleaser release --snapshot --clean` succeeds locally and in CI;
      artifacts for the 4 goreleaser targets + SHA256SUMS present; the
      windows zip is produced by its separate step (and its checksum is
      appended by that step).
- [ ] Untarred darwin/arm64 binary: `gitquest version` shows injected values.
- [ ] `.goreleaser.yaml` sets `release.draft: true` — every tag run produces
      a DRAFT release that a human flips public (enforces the "merge ≠
      release" rule). Verified by a scratch `v0.1.0-rc.1` tag run producing a
      draft, then deleted.
- [ ] release.yml passes actionlint; permissions minimal; SHA-pinned.
- [ ] `docs/releasing.md` complete incl. the manual repo-settings checklist.

## Validation

```sh
goreleaser check && goreleaser release --snapshot --clean
actionlint .github/workflows/release.yml
```

## Dependencies

- 01, 02.

## Non-goals

- Actual v1.0.0 publication (human gate); signing/SBOM (v2); homebrew tap
  activation (human).

## Design References

- DESIGN §14, §10.7; repo rules (merge ≠ release, secrets are human-managed).
