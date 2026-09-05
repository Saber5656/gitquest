# Title

CI workflow: lint, race tests, build matrix, govulncheck, SHA-pinned actions

## Summary

Add GitHub Actions CI enforcing the project's quality and supply-chain rules on
every push and pull request.

## Context

DESIGN §10.7 requires CI to be part of the security posture from the start:
pinned actions, minimal permissions, vulnerability scanning. CI must exist
before feature issues so every subsequent PR is gated.

## Scope

- `.github/workflows/ci.yml` with jobs:
  1. `lint`: `golangci-lint` (official action), `gofmt` check.
  2. `test`: `go test -race -count=1 ./...` on matrix
     `{ubuntu-latest, macos-latest} × Go {stable, oldstable}`.
  3. `build`: `CGO_ENABLED=0 go build ./cmd/gitquest` on the same OS matrix,
     plus `GOOS=windows GOARCH=amd64 go build` cross-compile check on ubuntu
     (build-only; failures in this step must be non-blocking via
     `continue-on-error: true` per DESIGN §2.3 Windows posture).
  4. `vuln`: `govulncheck ./...`.
- `.github/dependabot.yml`: weekly `gomod` + `github-actions` updates.

## Detailed Requirements

1. Every third-party action reference is pinned to a full commit SHA with a
   trailing version comment, e.g. `uses: actions/checkout@<sha> # v4.x.x`.
2. Top-level `permissions: contents: read`. No job may request more.
3. `concurrency` group cancels superseded runs per ref.
4. Go version selection via `actions/setup-go` with `go-version: stable` /
   `oldstable` (no hardcoded minor versions to maintain).
5. Workflow must not use `pull_request_target`, self-hosted runners, or any
   secret.
6. Add a `network-guard` step in `lint`: `! grep -rEn '"net/http"|"net"$|net\.Dial' --include='*.go' cmd/ internal/ || (echo "network import found" && exit 1)`
   — implementation may refine the pattern but MUST fail when `net/http` or
   `net.Dial` appears under `cmd/` or `internal/` (DESIGN I-2). Allow an
   explicit annotated exception list file `scripts/netguard-allow.txt`
   (empty in v1).

## Acceptance Criteria

- [ ] CI runs on `push` to any branch and on `pull_request`; all four jobs pass
      on the issue-01 skeleton.
- [ ] All actions SHA-pinned; `permissions` minimal; no secrets referenced.
- [ ] Windows cross-build step exists and is non-blocking.
- [ ] Network-guard step fails if a test file adds `net/http` under `internal/`
      (verified once by a scratch commit in the PR, then reverted).
- [ ] Dependabot config present for gomod + github-actions.

## Validation

Open the PR and observe: 4 green jobs; then push a scratch commit importing
`net/http` in `internal/cli` to see `lint` fail; revert. Attach both run links
to the PR description.

## Dependencies

- 01 (project skeleton must compile).

## Non-goals

- No release workflow (issue 47). No CodeQL/secret-scanning repo settings
  (manual repo-admin steps; document in issue 47's checklist instead).

## Design References

- DESIGN §10.7 (supply chain), §13 (CI matrix), §2.3 (Windows posture), I-2 (offline).
