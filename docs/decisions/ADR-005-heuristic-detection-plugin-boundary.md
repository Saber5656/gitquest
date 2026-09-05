# ADR-005: Language-agnostic heuristic detection in v1; external analyzers behind a plugin boundary in v2

Status: Accepted (user-confirmed, 2026-07-10)

## Context

"Dead code becomes monsters" needs a detection engine. True unused-symbol
analysis is language-specific and heavy (per-language ASTs or external tools
like knip/vulture/ts-prune, which require user-installed toolchains).

## Decision

- v1 ships only built-in, language-agnostic heuristics (six species —
  DESIGN §6.4): commented-out code blocks, fossilized TODOs, long-abandoned
  large files, leftover/backup artifacts, committed conflict markers,
  monolithic files. Zero configuration, works on any repository.
- Detection is defined behind a pure, I/O-free `Detector` interface
  (DESIGN §6.1). This interface IS the v2 plugin boundary; external-tool
  adapters (subprocess bridges) implement it later without pipeline changes.
- False positives are a designed-for case: every monster can be tamed
  (permanently ignored), and thresholds are config-tunable within clamps.

## Consequences

- Any repo is playable immediately; no toolchain detection matrix in v1.
- Heuristics will mislabel some living code — mitigated by tame, threshold
  clamps, curated default exclusions, and the playful framing (a "monster"
  claim is lighter than a "dead code" claim from a linter).
- The pure interface keeps detectors unit-testable with plain fixtures and
  forces the pipeline to own all I/O, budgets, and parallelism in one place.
