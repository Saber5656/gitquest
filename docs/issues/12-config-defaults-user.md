# Title

Configuration engine: built-in defaults, user config file, clamp table

## Summary

Implement the layered configuration system (defaults ← user config ← repo
config ← flags) with the security clamp table, exposing one resolved, typed
`Config` to the rest of the program. This issue covers defaults + user config
+ clamping; the untrusted repo-config loader is issue 13.

## Context

DESIGN §9 defines precedence, keys, defaults, and clamps. Clamps are a
security control (§9.3): resource budgets must stay bounded regardless of
config origin, and repo config must not be able to RAISE budgets.

## Scope

`internal/config/config.go`, `clamp.go`, `user.go` (+ tests):

```go
type Config struct {
    Scan struct {
        Exclude            []string
        IncludeOnly        []string
        UseDefaultExcludes bool
        MaxFiles           int
        MaxTotalBytes      int64
        SkeletonBlameBudget int
    }
    Thresholds struct {
        ZombieMinLines, SkeletonMinAgeDays, GhostMinAgeDays,
        GhostMinLines, GolemMinLines int
    }
    Monsters struct{ Disable []string }
    UI struct{ Theme string; Color string } // theme: auto|dark|light|mono; color: auto|always|never
}
func Defaults() Config

type Origin int // OriginDefault | OriginUser | OriginRepo | OriginFlag

type Warning struct{ Code, Message string } // shared warning shape for this package

// Layer is a sparse overlay: every field is a pointer (nil = unset), so an
// explicitly-empty list (e.g. disable = []) is distinguishable from absent.
type Layer struct {
    Origin Origin // who supplied this layer — drives budget-raise policy
    Scan struct {
        Exclude            *[]string
        IncludeOnly        *[]string
        UseDefaultExcludes *bool
        MaxFiles           *int
        MaxTotalBytes      *int64
        SkeletonBlameBudget *int
    }
    Thresholds struct {
        ZombieMinLines, SkeletonMinAgeDays, GhostMinAgeDays,
        GhostMinLines, GolemMinLines *int
    }
    Monsters struct{ Disable *[]string }
    UI struct{ Theme, Color *string }
}
func LoadUserLayer(path string) (Layer, []Warning, error) // TOML; Origin=OriginUser
func Resolve(base Config, layers ...Layer) (Config, []Warning) // reads each Layer.Origin
```

## Detailed Requirements

1. Defaults exactly per DESIGN §9.3 table (zombie 8, skeleton 365, ghost
   730/200, golem 1000, max_files 100000, max_total_bytes 512 MiB,
   blame budget 500) + `UseDefaultExcludes=true`, theme/color `auto`.
2. Clamping: after resolution, every numeric key is clamped to the §9.3
   min/max; a clamped value emits a Warning naming key, requested, and applied
   value. Budget keys (`max_files`, `max_total_bytes`, `skeleton_blame_budget`)
   accept raises ONLY from layers with `Layer.Origin ∈ {Default, User, Flag}`;
   a budget raise from an `OriginRepo` layer is ignored with a warning
   (lowering is allowed). Species `Disable` accepts only known species keys;
   unknown → warning, ignored.
3. TOML parsing via `BurntSushi/toml` with `toml.MetaData` to distinguish
   set/unset (sparse Layer). Unknown keys → Warning list (not error).
4. Missing user config file is not an error (empty layer). Malformed TOML →
   error (exit 1) — the user owns this file, fail loud.
5. `Exclude`/`IncludeOnly` patterns validated: must be relative, no `..`
   segment, no leading `/`, valid doublestar syntax; invalid → warning + drop
   (checked here so both user and repo layers share the rule).
6. Theme/color enums validated; invalid → warning + default.
7. No I/O outside `LoadUserLayer`; `Resolve` is pure (unit-test goldmine).

## Acceptance Criteria

- [ ] `Defaults()` matches the DESIGN table (golden test literal).
- [ ] Precedence test: default < user < repo < flag for a threshold key.
- [ ] Clamp tests: zombie_min_lines 2→3 and 999→200 with warnings.
- [ ] Repo-origin raise of `max_files` 100000→400000 ignored with warning;
      user-origin same raise accepted (≤ clamp max 500000).
- [ ] Unknown key `[thresholds] dragon_min=1` → warning containing `dragon_min`.
- [ ] Invalid pattern `../secrets/**` dropped with warning.
- [ ] Malformed user TOML → error; absent file → empty layer, no warning.

## Validation

```sh
go test -race ./internal/config/...
```

## Dependencies

- 01; 09 (config root path used by CLI wiring later; the loader itself takes a path).

## Non-goals

- Repo-config acquisition via blob read (13). Flag binding (30/32).

## Design References

- DESIGN §9.1, §9.3, §10.2; §6.3 (pattern semantics).
