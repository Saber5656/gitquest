# Title

Language comment map: extension → comment syntax table

## Summary

Implement the static language table mapping file extensions to comment
syntax (line markers, block pairs) and a `code_like` flag, per DESIGN §6.6.
Consumed by the Zombie detector (20) and Golem's code-blob rule (25).

## Scope

`internal/detect/langmap.go` (+ tests):

```go
type LangInfo struct {
    Name         string
    LineMarkers  []string   // e.g. ["//"], ["#"], ["--"]
    BlockPairs   [][2]string // e.g. [["/*","*/"]]
    CodeLike     bool       // false for Markdown/plain docs
    Known        bool
}
func LangForPath(path string) LangInfo // by extension (lowercased), basename fallbacks
```

## Detailed Requirements

1. Cover at minimum (DESIGN §6.6): Go, JavaScript, TypeScript, JSX/TSX, Python,
   Ruby, Rust, Java, Kotlin, C, C++, Objective-C (h/c/cc/cpp/hpp/m/mm), C#,
   PHP, Swift, Shell (sh/bash/zsh), PowerShell (ps1/psm1), SQL, HTML, CSS,
   SCSS, YAML (yml/yaml), TOML, Lua, Perl (pl/pm), Elixir (ex/exs), Haskell,
   Markdown (md/markdown, `CodeLike=false`). Add JSON (jsonc line comments
   only for `.jsonc`; `.json` has NO comment markers → Zombie never fires).
2. Extension table is a single package-level `map[string]LangInfo`; ambiguous
   `.h` maps to C (accepted approximation, comment in code). Basename
   fallbacks: `Makefile`→`#`, `Dockerfile`→`#`, `CMakeLists.txt`→`#`.
3. Unknown extension → `LangInfo{Known:false}` (zero markers). Detectors must
   treat unknown as "no comment analysis" (DESIGN §6.6).
4. Multi-marker languages: PHP `//` and `#`; SQL `--`; Lua `--`; HTML has
   block pair `<!-- -->` only (no line marker) — Zombie's v1 algorithm uses
   LINE markers only, so HTML effectively opts out; document this limitation
   in godoc (block-comment scanning is a v2 extension of issue 20).
5. Table completeness is a test: every listed language name above appears; a
   `TestLangTableFrozen` golden guards against accidental removals.

## Acceptance Criteria

- [ ] `LangForPath("a/b/x.go")` → Go with `//` + `/* */`, CodeLike.
- [ ] `.md` → CodeLike=false; `.json` → Known=true with zero markers;
      unknown `.xyz` → Known=false.
- [ ] Basename fallbacks work regardless of directory.
- [ ] Case-insensitivity: `X.GO` matches Go.
- [ ] Frozen-table test enumerates ≥ 30 extensions.

## Validation

```sh
go test -race ./internal/detect/ -run TestLang
```

## Dependencies

- 01.

## Non-goals

- Zombie block detection itself (20); content sniffing (shebang-based language
  detection is v2).

## Design References

- DESIGN §6.6, §6.4.1.
