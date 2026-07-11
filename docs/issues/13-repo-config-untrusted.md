# Title

Repo-local `.gitquest.toml` loader with untrusted-input hardening

## Summary

Read `.gitquest.toml` from the repository's HEAD tree (blob read, not
worktree), parse it defensively as UNTRUSTED input, and produce an
origin-tagged config layer (issue 12) that can never raise resource budgets
or escape the repository.

## Context

DESIGN §9.2 defines the schema and the 64 KiB limit; §10.1/§10.2 make
malicious repo config an explicit threat (DoS amplification, path escape).
A hostile repo must at worst make its own scan less interesting.

## Scope

`internal/config/repoconfig.go` (+ tests):

```go
// LoadRepoLayer reads .gitquest.toml from the HEAD tree via BlobReader.
// Absent file → empty layer, no warning. Every anomaly is a Warning, never a
// fatal error (a broken repo config must not block scanning).
func LoadRepoLayer(entries []gitio.TreeEntry, blobs *gitio.BlobReader) (Layer, []Warning)
```

## Detailed Requirements

1. Locate the entry with path exactly `.gitquest.toml` at repo root in the
   HEAD tree listing (no worktree fallback — uncommitted config is invisible,
   by design; document this in the code and in issue 48's docs).
2. Size guard BEFORE parse, using `TreeEntry.Size` (from `ls-tree --long`,
   issue 07) so no oversized content is ever buffered: size > 64 KiB →
   warning `repo config ignored: exceeds 64KiB`, empty layer, blob not read.
   The subsequent `BlobReader.Read` call passes `maxBytes = 64 KiB` as a
   second line of defense.
3. Parse with the same TOML machinery as issue 12 but with a
   recover-wrapped decode: any panic in the TOML library becomes a warning +
   empty layer (defense in depth for adversarial input).
4. Malformed TOML → warning + empty layer (contrast: user config fails loud;
   repo config fails soft — different trust posture, per DESIGN §9.2).
5. Only the keys in DESIGN §9.2 are read (`schema`, `[scan] exclude/
   include_only/use_default_excludes`, `[thresholds] five keys`,
   `[monsters] disable`). `schema` ≠ 1 → warning + empty layer. Unknown keys →
   warning (listing up to 10 key names, sanitized).
6. The produced Layer is tagged `OriginRepo`, which makes issue 12's Resolve
   reject budget raises automatically; this issue adds a belt-and-suspenders
   test proving budget keys present in repo TOML are dropped at load time
   (they are not even part of the §9.2 repo schema).
7. String values are length-capped (patterns ≤ 256 chars each, ≤ 100 patterns,
   species names ≤ 32 chars) — excess dropped with warning.
8. All warnings carry sanitized excerpts only (no raw control bytes — the
   render sanitizer (31) is applied downstream, but this loader must also cap
   excerpt length to 80 chars).

## Acceptance Criteria

- [ ] Happy path: fixture repo with valid `.gitquest.toml` adjusting
      `zombie_min_lines = 12`, `exclude = ["gen/**"]` → layer reflects both.
- [ ] Oversized (65 KiB) config blob → empty layer + the exact warning.
- [ ] Malformed TOML, `schema = 2`, unknown table `[net]`, the unknown repo
      key fixture `[scan]\nmax_files = 999999` (budget keys are not part of
      the §9.2 repo schema → unknown-key warning + drop), absolute pattern
      `/etc/**`, `..` pattern, 101 patterns, 300-char pattern → each produces
      its specified warning/drop behavior (table test).
- [ ] Config present only in worktree (not committed) → not loaded.
- [ ] Fuzz test (`go test -fuzz`, 30 s in CI-short mode) on the loader with
      arbitrary bytes → never panics, never returns error, always
      empty-or-valid layer.

## Validation

```sh
go test -race ./internal/config/ -run TestRepoLayer
go test -fuzz FuzzRepoLayer -fuzztime 30s ./internal/config/
```

## Dependencies

- 12, 07 (TreeEntry/BlobReader).

## Non-goals

- Any new config keys; write-back; worktree reads.

## Design References

- DESIGN §9.2, §9.3, §10.1, §10.2.
