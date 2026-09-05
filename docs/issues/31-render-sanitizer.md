# Title

Render layer: theme system and repo-string sanitizer (ANSI-injection defense)

## Summary

Implement `internal/render`: lipgloss theme registry (auto/dark/light/mono),
the security-critical sanitizer for repo-derived strings, and shared output
primitives (monster display line, XP bar, event lines, tables, path
middle-truncation) used by all commands.

## Context

Every repo- or git-derived string that reaches a terminal is an injection
vector (DESIGN §10.3). This issue creates the single sanitization choke point
plus the visual identity of the product (DESIGN §8.4). It blocks all
command-output issues (32–40).

## Scope

`internal/render/sanitize.go`, `theme.go`, `components.go` (+ tests):

```go
// Sanitize strips C0/C1 controls and all ESC (0x1b) sequences, including
// the full body of CSI/OSC sequences (not just the ESC byte).
func Sanitize(s string) string            // strict: no \n, no \t survive
func SanitizeMultiline(s string) string   // keeps \n AND \t, strips the rest

type Theme struct{ /* lipgloss styles: Title, XPBar, per-species colors, Event styles, Warn, Err, Muted */ }
func ResolveTheme(flagTheme string, noColor bool, isTTY bool) Theme

type Renderer struct{ Out, Err io.Writer; Th Theme; Width int }
func (r *Renderer) MonsterLine(m game.Monster) string  // "Grubmaw the Forgotten (Ghost, Lv. 4) — src/x.js:120-154 [68 HP]"
func (r *Renderer) XPBar(level, into, toNext int) string
func (r *Renderer) EventLine(e game.Event) string      // one styled line per event type
func (r *Renderer) Table(headers []string, rows [][]string, opts ...TableOpt) string
// TableOpt: WithPathColumn(i int) marks column i for middle-truncation-first.
// Default heuristic: a column whose header is exactly "ROOM" or "PATH".
func (r *Renderer) TruncPath(p string, max int) string // "src/…/api.js"
func (r *Renderer) Warnf(format string, a ...any)      // stderr, sanitized args
```

## Detailed Requirements

1. Sanitizer (DESIGN §10.3), implemented as a single-pass scanner (not
   chained regexes): remove bytes 0x00–0x08, 0x0B–0x1F, 0x7F, and C1
   0x80–0x9F raw bytes in invalid-UTF-8 contexts. On 0x1B, consume and drop
   the ENTIRE escape sequence: CSI (`ESC [` through its final byte
   0x40–0x7E), OSC (`ESC ]` through BEL or ST `ESC \`), and two-byte escapes
   — so the payload of an OSC title-set is removed, not just the ESC byte.
   PROVE via fuzz test that output never contains 0x1B or disallowed C0.
   Unicode bidi/zero-width controls (U+200B–U+200F, U+202A–U+202E,
   U+2066–U+2069) are replaced with `�` (path spoofing defense).
2. EVERY repo-derived string in components passes through Sanitize inside
   this package: monster names are generated (safe) but paths, evidence,
   branch labels, commit subjects are not. Components take raw strings and
   sanitize internally — callers cannot forget. Enforcement: a package test
   greps `internal/cli` + `internal/tui` (once they exist) for direct
   `fmt.Fprintf(app.Out` of variables — advisory; the real gate is the E2E
   injection test (44/46).
3. Theme resolution precedence (from 30): `mono` flag > `--no-color`/`NO_COLOR`
   > explicit theme > auto (isTTY + `lipgloss.HasDarkBackground`); non-TTY
   stdout ⇒ mono automatically (pipes get plain text).
4. Species colors (fixed): zombie=green, skeleton=white/bone, ghost=cyan,
   slime=yellow, mimic=magenta, golem=orange (256-color codes chosen for
   dark/light legibility; document hexes in code).
5. Layout: all components degrade at Width=80 (DESIGN §8.4); `Table` middle-
   truncates the PATH column first; XPBar width 24 cells
   (`[████████░░░░░░░░░░░░░░░░] Lv 11 · 2,310/4,120`).
6. `EventLine` covers ALL 13 event types (14) with distinct icons/prefixes
   (`☠` slain, `⚑` quest, `★` level-up, `👑` boss, … final glyph set fixed
   here; must render in mono theme too — ASCII fallbacks when mono:
   `[slain]`, `[quest]`, `[lvl]`, `[boss]`).
7. Number formatting: thousands separators for XP (`15,230`); durations as
   `3.2y`/`45d` (shared helpers).
8. No global state; Renderer is passed via App (30).

## Acceptance Criteria

- [ ] Fuzz test (`FuzzSanitize`, 30 s CI-short): output never contains 0x1B,
      C0 (except kept \n/\t in multiline), or bidi controls; idempotent
      (Sanitize(Sanitize(x))==Sanitize(x)).
- [ ] Golden tests: an ANSI-bomb path `evil-\x1b]0;pwned\x07.go` renders as
      `evil-.go` (the whole OSC sequence including its payload is dropped);
      a TODO evidence containing `\x1b[2J` renders with the full CSI sequence
      removed (assert byte-absence of 0x1b and of the sequence body).
- [ ] Theme matrix test: 4 themes × TTY/non-TTY → expected color/mono modes.
- [ ] All 13 event types have golden lines in dark AND mono themes.
- [ ] MonsterLine/XPBar/Table goldens at width 80 and 120.
- [ ] Non-TTY auto-mono proven via pipe test.

## Validation

```sh
go test -race ./internal/render/... 
go test -fuzz FuzzSanitize -fuzztime 30s ./internal/render/
```

## Dependencies

- 12 (theme config), 14 (types), 30 (flag plumbing contract).

## Non-goals

- bubbletea widgets (41); JSON encoding (39 — encoding/json handles escaping).

## Design References

- DESIGN §8.4, §10.3, §3.9 (display form), §11 (error style).
