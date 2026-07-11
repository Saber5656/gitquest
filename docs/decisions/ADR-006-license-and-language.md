# ADR-006: MIT license; English-only user-facing output in v1

Status: Accepted (user-confirmed, 2026-07-10)

## Context

The repository is intended to become a public OSS project. License and output
language shape adoption, contribution, and implementation cost.

## Decision

- License: MIT (LICENSE file at repo root; SPDX headers not required).
- All user-facing strings (CLI output, TUI, errors, docs) are English only.
  No i18n framework in v1; string literals live near their render sites.
  An i18n boundary (message catalog) is a v2 consideration and must not be
  speculatively built in v1.

## Consequences

- MIT maximizes adoption/sharing for a playful developer tool; patent-clause
  needs (Apache-2.0) were judged unnecessary for this codebase.
- English-only keeps v1 lean and matches the global OSS audience; the
  repository owner's own docs/PRs remain English per repo policy.
- Future i18n will require a string-extraction pass; accepted cost.
