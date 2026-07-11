# Title

User documentation: full README, field guide, config reference, FAQ

## Summary

Complete the user-facing documentation to release quality: the full README
(hero demo, quickstart, field guide), `docs/usage.md`, `docs/config.md`, and
the FAQ, per DESIGN §14.

## Context

Final issue — everything it documents must already exist. The README is the
product's storefront; the "playful + useful" personality (§1.3) must carry
through the writing.

## Scope

`README.md` (rewrite), `docs/usage.md`, `docs/config.md`, FAQ section,
demo recording assets under `docs/assets/`.

## Detailed Requirements

1. README structure (≤ 300 lines):
   - tagline + hero demo GIF (asciinema/vhs recording of scan → status → map
     on a fixture repo; the recording SCRIPT is committed
     (`docs/assets/demo.tape` for charmbracelet/vhs) so the GIF is
     regenerable; GIF committed as binary — acceptable, < 3 MiB);
   - install: release binaries + `sha256sum -c` verification, `go install`;
     Homebrew section reads exactly "Homebrew: coming soon" in v1 (the tap
     ships inert per issue 47; update the section when the human owner
     activates it);
   - quickstart: 4 commands with real output snippets (from goldens — copy
     from 44's fixtures so docs never drift from tests);
   - Monster Field Guide: the §6.4.7 table expanded with one flavor line +
     one "how to slay" line per species;
   - taming & false positives (§1.3 framing: "some monsters are friends");
   - CI usage: `gitquest report --json`, `--fail-on-monsters`, exit code 5;
   - privacy & security summary (§10.8: offline, read-only, what state
     contains, where it lives) + link to SECURITY.md;
   - v2 roadmap teaser (from DESIGN §2.4), contributing link.
2. `docs/usage.md`: every command with synopsis, flags, examples, exit codes
   (generated content must match `--help` goldens from 30 — a doc test greps
   both for drift on command names/flags).
3. `docs/config.md`: full key reference (user + repo config), the clamp
   table (§9.3), precedence, `.gitquest.toml` example for teams, the
   "repo config cannot raise budgets" security note.
4. FAQ (in README or docs/faq.md, ≥ 8 entries): "my code isn't dead!"
   (tame), "why is my stable file a Ghost", "monorepo too slow" (budgets,
   excludes), "does it phone home" (no, provably), "can it delete code for
   me" (never — I-1), "team XP sharing" (v2), "Windows?", "how is XP
   calculated" (formula table copy).
5. Language/tone: English; playful in flavor text, precise in reference
   sections; all CLI output samples must be real captured output, not
   hand-typed.
6. Cross-check pass: every DESIGN §8 user-visible behavior appears in
   usage.md; every config key in config.md (checklist in PR description
   mapping §8/§9 items → doc anchors).

## Acceptance Criteria

- [ ] README complete per structure above; renders correctly on GitHub
      (tables, GIF, anchors).
- [ ] Demo tape committed and regenerable (`vhs docs/assets/demo.tape`
      documented; CI does NOT regenerate — manual asset).
- [ ] usage.md ↔ `--help` drift test green.
- [ ] config.md contains every key from `config.Defaults()` (reflection-
      driven doc test).
- [ ] FAQ ≥ 8 entries incl. the five mandated topics.
- [ ] markdownlint clean; all internal links resolve (link-check script or
      lychee in CI, SHA-pinned, offline-mode for internal anchors only).

## Validation

```sh
npx markdownlint-cli2 'README.md' 'docs/**/*.md'
go test ./... -run 'TestDocsDrift|TestConfigDocs'
```

## Dependencies

- 30–43 (documented surface), 44 (golden snippets), 47 (install
  instructions), 03 (skeleton being replaced).

## Non-goals

- Website/GitHub Pages (v2); localized docs / i18n (deferred to v2 —
  ADR-006 fixes English-only output and docs for v1); video tutorials.

## Design References

- DESIGN §14, §1.3, §8, §9, §10.8, §2.4.
