# Title

Bootstrap Go module, project layout, Makefile, and lint configuration

## Summary

Create the initial Go project skeleton for GitQuest: module definition, directory
layout per DESIGN §4.1, a `Makefile` with the standard targets, `golangci-lint`
configuration, and a compilable `gitquest version` stub.

## Context

This is the first implementation issue. Everything else builds on this layout.
The repository currently contains only `README.md` and `docs/`.

## Scope

- `go.mod` with module path `github.com/Saber5656/gitquest`, Go 1.22 (or the
  latest stable at implementation time; record the choice in the PR).
- Directory skeleton with placeholder `doc.go` files (package comment only):
  `cmd/gitquest/`, `internal/cli/`, `internal/gitio/`, `internal/detect/`,
  `internal/game/`, `internal/state/`, `internal/config/`, `internal/render/`,
  `internal/tui/`.
- `cmd/gitquest/main.go` that calls `internal/cli.Execute()` and exits with its
  returned code.
- `internal/cli/root.go` minimal cobra root: name `gitquest`, short description
  "Turn your repository into an RPG dungeon", and a `version` subcommand printing
  `gitquest dev (none, unknown)` from injectable vars
  (`var version, commit, date = "dev", "none", "unknown"`).
- `Makefile` targets: `build`, `test` (`go test -race ./...`), `lint`
  (`golangci-lint run`), `vet`, `fmt-check` (gofmt diff, non-zero on dirty),
  `clean`.
- `.golangci.yml` enabling at least: `govet`, `staticcheck`, `errcheck`,
  `revive`, `gosec`, `misspell`, `gofmt`, `goimports`.
- `.gitignore` for Go (binary name `gitquest`, `dist/`, coverage files).
- `.editorconfig` (tabs for Go, 2-space YAML/TOML/MD).

## Detailed Requirements

1. Only `cobra` may be added as a dependency in this issue (charm/toml/doublestar
   arrive with the issues that use them). Run `go mod tidy`.
2. `internal/cli.Execute() int` must map errors to exit codes per DESIGN §11
   (for now: cobra usage errors → 2, anything else → 1, success → 0). Implement
   the mapping as a small exported helper `ExitCode(err error) int` with unit tests,
   so later issues extend it rather than reinvent it.
3. Package comment in each `doc.go` states the package's single responsibility
   (one sentence, taken from DESIGN §4.1).
4. `make build` produces `./gitquest` with `CGO_ENABLED=0`.
5. No network access at runtime; no `init()` side effects beyond cobra wiring.

## Acceptance Criteria

- [ ] `make build && ./gitquest version` prints `gitquest dev (none, unknown)` and exits 0.
- [ ] `./gitquest --help` exits 0; `./gitquest nonsense` exits 2 (not 1).
- [ ] `make test`, `make lint`, `make vet`, `make fmt-check` all pass on a clean checkout.
- [ ] `go.mod` module path is exactly `github.com/Saber5656/gitquest`.
- [ ] Directory layout matches DESIGN §4.1 (allowing placeholder-only packages).
- [ ] No dependencies beyond cobra (+its transitive deps).

## Validation

```sh
make build && ./gitquest version && ./gitquest --help
./gitquest nonsense; test $? -eq 2
make test lint vet fmt-check
go mod tidy && git diff --exit-code go.mod go.sum
```

## Dependencies

None (first issue).

## Non-goals

- No scan logic, no state, no git access, no styling (later issues).
- Do not create CI workflows (issue 02) or LICENSE/README content (issue 03).

## Design References

- DESIGN §4.1 (module layout), §11 (exit codes), ADR-001 (stack).
