# Title

Community and license files: LICENSE (MIT), README skeleton, CONTRIBUTING, SECURITY

## Summary

Add the OSS baseline files: MIT license, a README skeleton with the product
pitch and installation placeholder, contribution guidelines, and a security
policy.

## Context

The repository is public from day one. ADR-006 fixes MIT; DESIGN §10.7 requires
a security policy; §14 sketches the README's final shape (completed later in
issue 48 — this issue only creates the honest skeleton).

## Scope

- `LICENSE` — MIT, copyright line: `Copyright (c) 2026 Saber5656`.
- `README.md` — replace the current 2-line stub with: project tagline (keep the
  existing sentence), a "Status: under construction — v1 in development, see
  docs/ISSUE_PLAN.md" banner, concept paragraph (dungeon/XP/monsters in 5–8
  lines), the six-species table (name + one-line trigger, from DESIGN §6.4.7),
  quickstart placeholder (`go install github.com/Saber5656/gitquest/cmd/gitquest@latest`),
  privacy note (offline, read-only, state in home dir — DESIGN §10.8), and a
  link to `docs/DESIGN.md`.
- `CONTRIBUTING.md` — PR-only workflow (no direct pushes to main), CI must pass,
  conventional-ish commit style (imperative subject ≤ 72 chars), one issue per
  PR preferred, English for issues/PRs, `make test lint` before pushing.
- `SECURITY.md` — supported versions table (v1 latest only), private reporting
  via GitHub Security Advisories ("Report a vulnerability" button), 90-day
  coordinated disclosure default, explicit scope note: "scanning untrusted
  repositories is a supported use case; injection/DoS findings are in scope."

## Detailed Requirements

1. README stays under 120 lines in this issue (full docs are issue 48).
2. No badges that imply CI/coverage/releases that do not exist yet; a CI badge
   may be added only if issue 02 is already merged.
3. All files in English.
4. Do not add a CODE_OF_CONDUCT in this issue (owner decision deferred; note it
   in the PR description as an open question).

## Acceptance Criteria

- [ ] `LICENSE` is verbatim MIT with the specified copyright line.
- [ ] README contains: status banner, concept, species table, quickstart
      placeholder, privacy note, DESIGN link — and nothing describing
      unimplemented behavior as existing.
- [ ] CONTRIBUTING and SECURITY exist with the specified content.
- [ ] `LICENSE` content byte-matches the canonical MIT template (only the
      copyright line customized) so GitHub's license detection will classify
      it as MIT (post-merge detection is informational, not a gate).

## Validation

Manual review against this issue's Scope checklist; run a markdown linter
(`npx markdownlint-cli2 '*.md'` or editor equivalent) — no errors.

## Dependencies

- 01 (repo skeleton exists; README references docs/ paths).

## Non-goals

- Final README with GIFs/FAQ (issue 48). Homebrew instructions (issue 47/48).

## Design References

- ADR-006 (MIT), DESIGN §10.7 (SECURITY.md), §10.8 (privacy note), §14 (README plan).
