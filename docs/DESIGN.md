# GitQuest — v1 Design Document

Status: Draft for v1 implementation
Canonical source of truth for requirements, architecture, and behavior.
Issue planning derived from this document lives in `docs/ISSUE_PLAN.md` and `docs/issues/*.md`.

---

## 1. Product Overview

### 1.1 Concept

GitQuest turns any git repository into an RPG dungeon:

- The directory tree is the dungeon. Top-level directories are floors; nesting is depth.
- Commits earn experience (XP). The player levels up by committing.
- Dead and decaying code manifests as monsters (zombies, skeletons, ghosts, slimes, mimics, golems).
- Deleting or fixing that code slays the monster and grants bonus XP.
- Quests point the player at the most valuable monsters to slay. The biggest pile of debt is the dungeon boss.

GitQuest is a fun skin over a genuinely useful activity: finding and deleting dead code.

### 1.2 Target users

- Individual developers who want a playful motivator to clean up their own repositories.
- Teams that want a light-hearted "tech debt radar" (shared via the repo-local config; shared progression is v2).
- CI users who want a machine-readable debt report (`gitquest report --json`).

### 1.3 Product personality (confirmed decision)

Playful-but-useful. RPG flavor is front and center in all human-facing output, but:

- Every monster maps to a concrete, actionable finding (file, line span, reason).
- The slay report doubles as a deletion-candidate list.
- False positives are a designed-for case: any monster can be "tamed" (ignored) and never reported again.

### 1.4 Platform & distribution

- Language: Go (single static binary). See ADR-001.
- Primary platforms: macOS and Linux (amd64/arm64). Windows: best effort, not release-blocking in v1.
- Distribution: GitHub Releases binaries + Homebrew tap (v1), `go install` always works.
- License: MIT. See ADR-006.

### 1.5 Hard product invariants (confirmed decisions)

These are non-negotiable constraints for every v1 issue:

| # | Invariant |
|---|---|
| I-1 | GitQuest NEVER writes to the target repository: no files, no refs, no config, no locks. All state lives under the user's home data directory. |
| I-2 | GitQuest is fully offline: no network I/O of any kind (no telemetry, no update checks, no uploads). |
| I-3 | All human-facing output is English (i18n boundary is a v2 concern). |
| I-4 | Repository content is untrusted input. See §10 Security Model. |
| I-5 | Player state model is solo in v1 but the schema must not prevent per-author aggregation later (see §7.2). |

---

## 2. Requirements

### 2.1 Confirmed product decisions

| Decision | Choice | Where recorded |
|---|---|---|
| Core form | Non-interactive CLI core + lightweight interactive `explore` mode; full TUI game deferred to v2 | ADR-003 |
| Detection engine | Built-in language-agnostic heuristics + a plugin boundary (interface only) for external analyzers in v2 | ADR-005 |
| State storage | Per-user home directory, keyed by repository identity; target repo never written | ADR-004 |
| Stack | Go + cobra + lipgloss + bubbletea | ADR-001 |
| Git access | Shell out to system `git` (read-only invocations), not go-git | ADR-002 |
| Personality | Playful + useful; ignore/tame is first-class | §1.3 |
| Player model | Solo; whole history = the player's adventure log; author-extensible data model | §3.4, §7.2 |
| v1 mechanics | XP/levels, bestiary, slaying, achievements, quests, boss | §3 |
| License | MIT | ADR-006 |
| UI language | English only | §1.5 I-3 |
| Network posture | Fully offline | §1.5 I-2 |

### 2.2 v1 scope

- Commands: `scan`, `status`, `map`, `monsters`, `quests`, `achievements`, `tame`, `report`, `reset`, `explore`, `version` (§8).
- Six monster species: Zombie, Skeleton, Ghost, Slime, Mimic, Golem (§6.4) + boss promotion (§3.7).
- XP from commit history, level curve, level-up events (§3.4).
- Slay/fled/tamed reconciliation across scans (§3.5, §6.5).
- Quests (up to 3 active suggestions) with completion bonus (§3.6).
- 15 achievements (§3.8).
- Repo-local `.gitquest.toml` (read-only, untrusted) and user config (§9).
- JSON report with a stable schema + `--fail-on-monsters` for CI (§8.3).
- Lightweight interactive `explore` mode: browse map/bestiary/quests/status, tame from detail view (§8.5).
- Release pipeline with checksummed artifacts (§14).

### 2.3 v1 non-goals

- No per-author parties, leaderboards, or multiplayer (data model ready, feature deferred).
- No external analyzer integrations (knip, vulture, …) — only the interface exists.
- No full-screen roguelike game; `explore` is a browser, not a game loop.
- No i18n/l10n machinery.
- No writes to the repository, ever (not a non-goal that can be lifted in v2 without a new ADR).
- No network features (badges, gist export, update check).
- No language-specific static analysis (AST-level unused-symbol detection).
- No history rewriting awareness beyond §5.4 (e.g., no reflog mining).
- No Windows release gating: build it, test best-effort, do not block v1.

### 2.4 v2 deferred ideas

- External detector plugins over the `Detector` boundary (knip/vulture/ts-prune adapters).
- Per-author party view (`--by-author`), shared team dungeons.
- Full TUI game mode (walk the dungeon, battles, inventory).
- Classes/items/streaks; seasonal events.
- Badge/JSON export for sharing; shields.io endpoint file generation.
- i18n.
- `gitquest watch` daemon / shell prompt integration.

### 2.5 Known unknowns (may spawn new issues during implementation)

| # | Unknown | Current plan |
|---|---|---|
| U-1 | False-positive rate of Zombie/Ghost heuristics on real repos | Tune thresholds against fixture corpus in issue 41; thresholds are config-overridable |
| U-2 | `git blame` cost on large repos for Skeleton | Cap blamed files per scan (§6.4.2); revisit with a persistent line-age cache if too slow |
| U-3 | Windows terminal rendering (lipgloss/bubbletea quirks) | Best-effort; mono theme fallback |
| U-4 | Shallow/partial clones | Detect via `git rev-parse --is-shallow-repository`; degrade (§5.5) |
| U-5 | Very large monorepos (>100k files) | Enforced budgets + skip reporting (§10.5); explicit unsupported note |
| U-6 | `git` version spread (need ≥ 2.30) | Version probe at startup; hard error below floor (§5.2) |

---

## 3. Game Design

### 3.1 Core loop

1. Player runs `gitquest scan` in a repository.
2. First scan: entire commit history is converted to XP ("recovered adventure log"), dungeon is generated, monsters appear.
3. Player reads `quests`/`map`/`monsters`, picks a target, deletes/fixes dead code in their normal workflow, commits.
4. Next `scan` detects the change: XP for new commits, slay events with bonus XP, quest completions, achievements, level-ups.
5. Repeat until `Dungeon Cleared` (zero living monsters) — or forever, as new debt spawns new monsters.

### 3.2 Dungeon mapping

| Repo concept | Dungeon concept |
|---|---|
| Repository | The dungeon |
| Top-level directory (and root files) | Floor (root files = "Entrance Hall") |
| Nested directory depth `d` | Dungeon depth `d` |
| File | Room |
| Detected finding | Monster in that room |
| Largest-HP monster in repo | Dungeon Boss |
| Largest-HP monster per floor (HP ≥ 100) | Floor Guardian |

### 3.3 Monsters

A monster is a stable, identifiable finding produced by a detector (§6.4). Common attributes:

| Field | Meaning |
|---|---|
| `id` | Stable identity across scans (§6.5) |
| `species` | `zombie` \| `skeleton` \| `ghost` \| `slime` \| `mimic` \| `golem` |
| `name` | Deterministically generated fantasy name (§3.9) |
| `path` | Repo-relative file path (the room) |
| `span` | Optional line range `[start, end]` (1-based, inclusive) |
| `hp` | Debt magnitude; also the slay XP bonus (§6.4 per-species formulas) |
| `level` | `clamp(floor(hp / 20) + 1, 1, 99)` |
| `status` | `alive` \| `slain` \| `fled` \| `tamed` |
| `evidence` | Human-readable one-line reason (e.g. "34-line commented-out code block") |
| `first_seen` / `resolved_at` | Timestamps + commit SHAs |

### 3.4 XP and levels

XP sources (evaluated per commit, in first-parent order over the scanned range):

| Source | XP | Notes |
|---|---|---|
| Any non-merge commit | 10 | Base reward |
| Churn bonus | `floor(10 * log10(1 + insertions + deletions))`, capped at 40 | Log scale defeats XP farming by giant commits |
| Purge bonus | +5 when `deletions > insertions` and `deletions ≥ 10` | Rewards net deletion |
| Merge commit | 5 flat | No churn/purge bonus (numstat across merges is misleading) |
| Slay bonus | `monster.hp`; when the slay completes an active quest, an additional `round(0.25 × hp)` quest bonus (math.Round, half away from zero) — total exactly `round(1.25 × hp)` | Credited at reconciliation time (§3.5); e.g. hp 50 → 50 + 13 = 63 XP |

Level curve (cumulative threshold; per-term: multiply by 100 FIRST, then floor):

```
xp_to_reach(L) = Σ_{k=1}^{L-1} floor(100 * k^1.5)       (L ≥ 1; xp_to_reach(1) = 0)
level(xp)      = max L such that xp ≥ xp_to_reach(L)    (hard cap L = 999)
```

Exact reference points: L2 = 100, L3 = 382, L4 = 901, L5 = 1,701, L10 = 11,102. At ≈ 15 XP/commit, a 1,000-commit history lands ≈ L11; a 50k-commit monorepo ≈ L51.

First scan behavior: all historical commits are converted to XP in one pass, emitting a single `first_scan_completed` event ("Recovered adventure log: N commits, +X XP, you are level L") instead of thousands of individual events. The player is solo: all commits count regardless of author (confirmed decision). The XP ledger records per-author subtotals internally to keep v2 party mode possible (§7.2), but v1 surfaces only the solo total.

### 3.5 Slaying rules

Reconciliation happens during `scan` by comparing the previous monster set with the fresh detection set (matching by `id`, §6.5):

| Transition | Condition | Effect |
|---|---|---|
| `alive → slain` | Monster no longer detected AND ≥ 1 scanned commit touched `path` (or deleted it) since the last scan | `monster_slain` event; slay bonus XP; quest completion check |
| `alive → fled` | Monster no longer detected but NO scanned commit touched its path (threshold drift, config change, detector change) | `monster_fled` event; no XP (prevents farming by toggling thresholds) |
| `alive → tamed` | Player ran `tame` on it | No XP; excluded from future reports; kept in bestiary as "tamed" |
| `alive → alive` | Still detected | HP/evidence refreshed; identity preserved |
| (new) `→ alive` | Newly detected | `monster_appeared` event (suppressed in bulk on first scan) |

Slain/fled/tamed monsters are retained in state (bestiary history) but never re-reported as findings. If the same finding reappears later (same identity), it is resurrected as a new `alive` monster with a `monster_appeared` event and its kill history intact.

### 3.6 Quests

- After each scan, up to 3 quests are active. Selection from living monsters: order by HP descending, at most one per species, skip tamed; refill only when a slot is empty (quests persist across scans until resolved).
- A quest is satisfied when its target monster becomes `slain` → `quest_completed` event, slay bonus ×1.25 (rounded) instead of ×1.0.
- If the target flees or is tamed, the quest is cancelled (`quest_cancelled` event) and the slot refills next scan.
- No acceptance step in v1: quests are standing bounties, not contracts.

### 3.7 Boss

- Dungeon Boss: the living monster with maximum HP, ties broken by earlier `first_seen`, then lexicographic path, then id. Recomputed every scan; promotion emits `boss_appeared` — including on the first scan (it is a single event, exempt from the first-scan bulk suppression of §3.5).
- Slaying it emits `boss_slain` (in addition to `monster_slain`).
- Floor Guardians: per top-level directory, the max-HP living monster with HP ≥ 100 (display flavor only; no extra XP).

### 3.8 Achievements (v1 set — exactly these 15)

| ID | Name | Unlock condition |
|---|---|---|
| `first-scan` | Night Watch | Complete first scan |
| `first-blood` | First Blood | First monster slain |
| `zombie-hunter` | Zombie Hunter | First zombie slain |
| `skeleton-crusher` | Skeleton Crusher | First skeleton slain |
| `ghost-buster` | Ghost Buster | First ghost slain |
| `slime-splitter` | Slime Splitter | First slime slain |
| `mimic-breaker` | Mimic Breaker | First mimic slain |
| `golem-toppler` | Golem Toppler | First golem slain |
| `boss-slayer` | Boss Slayer | Slay a dungeon boss |
| `exterminator` | Exterminator | 10 monsters slain (lifetime, this repo) |
| `dungeon-cleared` | Dungeon Cleared | A scan ends with 0 living monsters (≥ 1 lifetime slay) |
| `purifier` | Purifier | Lifetime deletions ≥ 10,000 lines (scanned commits) |
| `centurion` | Centurion | 100 commits scanned (lifetime, this repo) |
| `deep-delver` | Deep Delver | Slay a monster at directory depth ≥ 6 |
| `level-10` | Seasoned Adventurer | Reach level 10 |

Achievements unlock once, are timestamped, and emit `achievement_unlocked`.

### 3.9 Deterministic naming

Monster display names are generated from `sha256(monster.id)` via syllable tables (no RNG at runtime; same monster ⇒ same name on every machine). Format: `<GivenName> the <Epithet>` where epithet derives from species + HP tier, e.g. `Grubmaw the Forgotten (Ghost, Lv. 12)`. Tables live in `internal/game/naming.go`; ≥ 4,096 distinct given names; collision across different ids is acceptable (id remains the true key).

### 3.10 Events

Events are the single narrative channel; every command that changes state derives its human output from events. Event types (closed set for v1):

`first_scan_completed`, `xp_gained` (aggregated per scan), `level_up`, `monster_appeared`, `monster_slain`, `monster_fled`, `monster_tamed`, `quest_issued`, `quest_completed`, `quest_cancelled`, `achievement_unlocked`, `boss_appeared`, `boss_slain`.

The last 200 events are persisted (ring buffer) for `status` recap and `explore`.

---

## 4. System Architecture

### 4.1 Module layout

```
gitquest/
├── cmd/gitquest/main.go          # entry point; wires CLI
├── internal/cli/                 # cobra commands (one file per command)
│   ├── root.go                   # global flags, exit-code mapping, version
│   ├── scan.go status.go map.go monsters.go quests.go
│   ├── achievements.go tame.go report.go reset.go explore.go
├── internal/gitio/               # ALL git subprocess access (read-only)
│   ├── runner.go                 # exec wrapper, arg allowlist, timeouts
│   ├── repo.go                   # discovery, identity, preflight checks
│   ├── history.go                # commit stream (numstat, incremental)
│   ├── tree.go                   # HEAD tree listing + blob content via cat-file
│   └── blame.go                  # line-age queries
├── internal/detect/              # detection engine
│   ├── detector.go               # Detector interface (plugin boundary)
│   ├── pipeline.go               # orchestration, budgets, parallelism
│   ├── langmap.go                # extension → comment syntax table
│   ├── exclusions.go             # default path/generated-file exclusions
│   ├── zombie.go skeleton.go ghost.go slime.go mimic.go golem.go
├── internal/game/                # pure game logic (no I/O)
│   ├── types.go                  # Monster, Player, Quest, Achievement, Event
│   ├── xp.go level.go            # formulas (§3.4)
│   ├── reconcile.go              # slay/fled/tame matching (§3.5, §6.5)
│   ├── quest.go achievement.go boss.go naming.go
├── internal/state/               # persistence
│   ├── paths.go                  # data-dir resolution, repo-id → profile dir
│   ├── store.go                  # load/save atomic, locking, recovery
│   ├── schema.go                 # versioned state structs + migration hooks
│   └── cache.go                  # derived caches (file last-touched index)
├── internal/config/              # defaults, user config, repo config (untrusted)
├── internal/render/              # lipgloss styles, text/JSON renderers, sanitizer
├── internal/tui/                 # explore mode (bubbletea)
└── docs/                         # this document, ADRs, issue plan
```

Dependency rule (enforced by review + `internal/` layout): `cli → {gitio, detect, game, state, config, render, tui}`; `detect → {gitio(read interfaces), config, game(types)}`; `game` imports nothing but stdlib; `render` imports `game` types only; `tui → {game, state(read), render}`. No package imports `cli`.

### 4.2 Scan pipeline (sequence)

```
scan
 1. gitio.repo:    discover root (rev-parse), preflight (git version, bare?,
                   shallow?, empty?), compute repo identity (§7.1)
 2. state.store:   acquire profile lock; load state (or init); load caches
 3. config:        merge defaults ← user config ← repo .gitquest.toml ← flags
 4. gitio.history: stream commits last_scan..HEAD (first scan: all), first-parent
 5. game.xp:       fold commits → XP delta, per-author ledger, purge stats
 6. gitio.tree:    list HEAD tree; classify (binary/size/excluded); build
                   candidate file set; update last-touched cache from step 4
 7. detect:        run detectors over candidates (worker pool, budgets)
 8. game.reconcile: match old vs new monsters → slain/fled/appeared
 9. game:          quests refill/complete, boss recompute, achievements, level-ups
10. state.store:   persist state + caches (atomic), release lock
11. render:        emit events + summary (text or JSON)
```

Failure in steps 1–3 exits with the mapped code (§11) before any state change. Failures in 4–9 abort without persisting (state stays at previous scan; lock released via defer).

### 4.3 Concurrency

- Single process assumption per profile enforced by an exclusive advisory lock file (§7.3); second concurrent `scan` fails fast with exit 1 and a friendly message.
- Detector execution: worker pool, `min(GOMAXPROCS, 8)` workers; each file is processed by all applicable detectors in one pass over its content.

---

## 5. Git Access Layer

### 5.1 Read-only guarantees

- Every git invocation goes through `gitio.Runner`, which only permits a fixed allowlist of subcommands: `version`, `rev-parse`, `rev-list`, `log`, `ls-tree`, `cat-file`, `blame`, `diff-tree`. Anything else panics in tests / errors at runtime.
- All invocations pass `--no-optional-locks` and set `GIT_OPTIONAL_LOCKS=0` (prevents even benign lock writes).
- Hardened environment: subprocess env is constructed from scratch: `PATH`, `HOME`, `LC_ALL=C`, `GIT_TERMINAL_PROMPT=0`, `GIT_CONFIG_NOSYSTEM=0` (system config allowed), plus `GIT_OPTIONAL_LOCKS=0`. Never forward `GIT_DIR`, `GIT_WORK_TREE`, `GIT_INDEX_FILE` from the parent environment (prevents redirection attacks); set `GIT_DIR` explicitly to the discovered repo.
- `exec.Command` direct (argv array); never through a shell. Paths always separated with `--`.
- Worktree files are NEVER opened directly: all content reads go through `git cat-file` blobs (§5.3). This makes symlink escape and TOCTOU on worktree files structurally impossible (§10.4).

### 5.2 Exact git usage

Preflight:

| Purpose | Command |
|---|---|
| Version floor (≥ 2.30) | `git version` |
| Repo root | `git rev-parse --show-toplevel` |
| Bare check (reject, exit 3) | `git rev-parse --is-bare-repository` |
| Shallow check (degrade, §5.5) | `git rev-parse --is-shallow-repository` |
| HEAD (empty repo → exit 3) | `git rev-parse --verify HEAD` |
| Repo identity | `git rev-list --max-parents=0 HEAD` (all roots; pick the lexicographically smallest SHA) |

History (step 4): one streamed process —
`git log --first-parent --reverse --date=unix -z --numstat --format=%x01%H%x00%P%x00%at%x00%aN%x00%aE <range>`
where `<range>` is `HEAD` on first scan, else `<last_scan_sha>..HEAD`. Parser contract, including rename lines (`-M` not passed ⇒ numstat emits plain add/delete; rename tracking instead uses `diff-tree`, below) is specified in issue 05. If `last_scan_sha` is no longer reachable (rebase/force-push upstream), fall back to full re-scan of XP with an informational warning and keep lifetime counters monotonic (never subtract XP).

Per-commit touched paths for reconciliation (step 8) come from the same `--numstat` stream (no extra process). Renames for monster identity migration use one extra call per scan over the whole range: `git diff-tree -r -z -M50 --name-status <last_scan_sha> HEAD` (first scan: skipped).

Tree snapshot (step 6): `git ls-tree -r -z --long HEAD` → `(mode, type, sha, size, path)`. Filter: keep `blob` entries with mode `100644`/`100755`; skip `120000` (symlinks), `160000` (submodules; also emit one aggregate note in scan warnings).

Content: `git cat-file --batch` long-lived subprocess; request only candidate blobs (post-exclusion, size ≤ 2 MiB); enforce read budget (§10.5).

Blame (Skeleton only): `git blame --porcelain -w HEAD -- <path>`, capped (§6.4.2).

### 5.3 Incremental scan algorithm

State keeps `last_scan{sha, at}`. Scan cost is `O(new commits + changed files + detector pass over candidate files)`. The last-touched-per-file cache (`cache.json`) is built on first scan from the full history stream and updated incrementally from new commits; Ghost/Skeleton detectors consume it instead of running `git log` per file (which would be O(files × history)).

### 5.4 History rewrites

If `last_scan_sha` is unreachable from HEAD: warn (`history rewritten — recovering`), rebuild caches from full history, recompute monsters normally, do NOT re-grant historical XP (lifetime XP only ever grows; the ledger stores `total_commits_scanned` and awards XP only for commits with timestamp/SHA not already folded — approximated by counting first-parent commits reachable from HEAD but not previously counted; exact rule in issue 05).

### 5.5 Degraded modes

| Condition | Behavior |
|---|---|
| Shallow clone | XP from available commits only; Ghost/Skeleton age data may be truncated → warning banner `⚠ shallow clone: dungeon history is incomplete` |
| Detached HEAD | Fully supported (identity uses root commit, not branch) |
| Empty repo (no HEAD) | Exit 3: `This dungeon has no history yet — make your first commit.` |
| Bare repo | Exit 3 (no worktree semantics needed, but out of v1 scope) |
| Not a git repo | Exit 3 with hint |

---

## 6. Detection Engine

### 6.1 Detector interface (the v2 plugin boundary)

```go
// internal/detect/detector.go
type FileContext struct {
    Path        string   // repo-relative, slash-separated
    Size        int64    // blob size in bytes
    Lines       []string // decoded content lines (nil if size/binary-skipped)
    IsBinary    bool
    Truncated   bool     // size exceeded the per-file budget: Lines nil, Size valid
    Depth       int      // directory depth; root file = 1
    Lang        LangInfo // from langmap; may be Unknown
    LastTouched int64    // unix ts of last commit touching path (cache)
    RepoNow     int64    // timestamp of HEAD commit (not wall clock)
    RepoAlive   bool     // ≥1 commit repo-wide in the last 90 days (Ghost gate, §6.4.3)
    LineAges    []int64  // per-line author times, index 0 = line 1; non-nil ONLY when
                         // the detector implements AgeRequester and blame budget allowed
}

type Finding struct {
    Species   string // stable species key
    Path      string
    Span      *[2]int // optional [startLine, endLine], 1-based inclusive
    HP        int
    Evidence  string  // one-line human-readable reason (sanitized later)
    Fingerprint string // species-specific content fingerprint (§6.5)
}

type Detector interface {
    Species() string
    // Examine returns zero or more findings for one file. Must be pure,
    // deterministic, and must not perform I/O (all inputs precomputed).
    Examine(ctx FileContext) []Finding
}

// AgeRequester is an optional capability interface. The pipeline calls
// WantsAges (a cheap prematch, e.g. a TODO regex) after building the base
// FileContext; when it returns true and the blame budget permits, the
// pipeline fills ctx.LineAges before calling Examine. Examine stays pure.
type AgeRequester interface {
    WantsAges(ctx FileContext) bool
}
```

External analyzer adapters in v2 will implement `Detector` behind a subprocess bridge; the interface is intentionally I/O-free so v1 detectors stay trivially testable. The only detector-triggered extra pass in v1 is blame, mediated by `AgeRequester` + `FileContext.LineAges` (Skeleton, issue 21); the pipeline owns the I/O and the budget.

### 6.2 Pipeline & budgets

1. Candidate set = HEAD tree blobs minus exclusions (§6.3) minus binaries (NUL byte in first 8 KiB or invalid UTF-8 > 10% of first 8 KiB) minus files > 2 MiB (except Golem-by-size, §6.4.6).
2. Budgets (defaults; overridable in config within clamps §9.3): max 100,000 candidate files; max 512 MiB total bytes decoded; max 500 blamed files; per-git-call timeout 120 s; whole scan timeout 10 min. Exceeding a budget skips remaining work of that kind and records a scan warning (`skipped: N files over byte budget`) — never silently.
3. Worker pool fans out per file; each worker runs all applicable detectors over the decoded lines once.
4. Findings → reconciliation (§6.5).

### 6.3 Default exclusions

Path prefixes/globs (case-sensitive, repo-relative, `doublestar` semantics), each applied at BOTH the root and any nesting depth (i.e. `vendor/**` AND `**/vendor/**`): `vendor/`, `node_modules/`, `third_party/`, `dist/`, `build/`, `out/`, `target/`, `.git/` (defensive), `testdata/`, plus `**/*.min.*` and lockfile basenames at any depth (`go.sum`, `package-lock.json`, `yarn.lock`, `pnpm-lock.yaml`, `Cargo.lock`, `poetry.lock`, `Gemfile.lock`, `composer.lock`).
Generated-file check: first 16 lines matching `(?i)(code generated .* do not edit|@generated|autogenerated file)` ⇒ excluded.
User/repo config can extend `exclude` and narrow via `include_only` (§9.2); defaults always apply unless `use_default_excludes = false`.

### 6.4 Species specifications

Shared notation: `age_days(x) = floor((RepoNow - x) / 86400)`. All thresholds are the config keys in §9.3.

#### 6.4.1 Zombie — commented-out code block

- Applicability: files whose language has a known line-comment marker (§6.6).
- Algorithm: scan for maximal runs of ≥ `zombie_min_lines` (default 8) consecutive lines that are line-comments (after stripping indentation). For the run, compute `codeness` = fraction of lines matching any of: ends with `;`, `{`, `}`, `)`, `):`; contains `=` not inside prose; matches keyword regex `\b(if|else|for|while|return|func|def|class|import|switch|case|try|catch|end)\b`. Finding iff `codeness ≥ 0.4`.
- False-positive guards: skip runs starting within the first 20 lines that contain `(?i)(copyright|license|permission)`; skip runs where > 50% of lines end with `.` or start with `#`/`##` (markdown-ish prose); skip doc-comment syntax (`///`, `//!`, `/** … */` block openers) when the run is ≥ 60% prose-like.
- HP: `min(2 * run_lines, 500)`. Span: the run. Fingerprint: sha256 of the run's normalized text (whitespace-collapsed), first 16 hex chars.

#### 6.4.2 Skeleton — fossilized TODOs

- Applicability: any candidate text file containing `\b(TODO|FIXME|HACK|XXX)\b` outside of the exclusion guards.
- Line ages come from one `git blame --porcelain -w HEAD -- <path>` per candidate file (only files that matched the regex), capped at `skeleton_blame_budget` (default 500 files/scan, largest files first are NOT prioritized — order by match count descending). Over-budget files are skipped with a scan warning.
- A TODO line is a "bone" iff `age_days(line_author_time) ≥ skeleton_min_age_days` (default 365).
- One Skeleton per file (bones aggregated). HP: `min(15 * bones + 10 * floor(max_bone_age_days / 365), 400)`. Span: `[first_bone_line, last_bone_line]`. Fingerprint: sha256 over sorted bone texts (normalized), first 16 hex.

#### 6.4.3 Ghost — long-abandoned large file in a living repo

- Preconditions: repo is alive (≥ 1 commit in the last 90 days across the whole repo) — otherwise Ghost detection is disabled for the scan (a dead repo is a mausoleum, not a haunted dungeon).
- Finding iff `age_days(LastTouched) ≥ ghost_min_age_days` (default 730) AND `len(Lines) ≥ ghost_min_lines` (default 200).
- HP: `min(floor(lines / 10) * clamp(floor(age_days / 365), 1, 5), 600)` — the age factor is floored at 1 so a lowered `ghost_min_age_days` config can never produce a 0-HP finding. Span: none (whole file). Fingerprint: `"file"` (identity is path+species).

#### 6.4.4 Slime — backup/leftover artifacts

- Filename patterns (basename, case-insensitive): `*.bak`, `*.orig`, `*.old`, `*.rej`, `*~`, `*.save`, `*.swp`, `*.tmp`, `copy of *`, `* copy.*`, `*_old.*`, `*_backup.*`, `* (conflicted copy*`.
- Applies to any tracked blob (including binary; content not read). HP: `min(10 + floor(size_bytes / 1024), 100)`. Fingerprint: `"file"`.

#### 6.4.5 Mimic — committed merge-conflict markers

- Text files only. Finding iff the file contains a complete marker set: a line starting `<<<<<<< `, then a line exactly `=======`, then a line starting `>>>>>>> `, in that order (multiple sets allowed).
- Guard: all three markers must appear at column 0; files under `docs/**` with fenced code blocks containing the markers still match (acceptable; tame exists).
- HP: `min(40 * marker_sets, 200)`. Span: first set's range. Fingerprint: sha256 of joined marker-line numbers + branch labels, first 16 hex.

#### 6.4.6 Golem — monolithic file

- Text file with `len(Lines) ≥ golem_min_lines` (default 1000), OR any blob with `size > 2 MiB` (content-skipped) and extension in the langmap (code-like) — the latter uses size-based HP.
- HP: `min(floor(lines / 20), 800)`; size-based variant (Truncated code blobs) `min(floor(size_bytes / 1600), 800)` — one HP per 1,600 bytes ≈ 20 lines at 80 B/line, aligning both paths. Fingerprint: `"file"`.

#### 6.4.7 Species/threshold summary

| Species | Trigger (defaults) | HP (cap) | Identity granularity |
|---|---|---|---|
| Zombie | ≥ 8-line commented-out code run, codeness ≥ 0.4 | `2×lines` (500) | path + span fingerprint |
| Skeleton | ≥ 1 TODO/FIXME/HACK/XXX aged ≥ 365 d | `15×bones + 10×years` (400) | path + bone fingerprint (one per file) |
| Ghost | file untouched ≥ 730 d ∧ ≥ 200 lines ∧ repo alive | `lines/10 × clamp(years,1,5)` (600) | path |
| Slime | leftover filename pattern | `10 + KiB` (100) | path |
| Mimic | complete conflict-marker set | `40×sets` (200) | path + marker fingerprint |
| Golem | ≥ 1000 lines (or > 2 MiB code blob) | `lines/20` (800) | path |

### 6.5 Monster identity & reconciliation

- `id = species + ":" + path + ":" + fingerprint` (fingerprint is the literal `"file"` for the whole-file species ghost, slime, and golem; zombie/mimic/skeleton use their content fingerprints from §6.4).
- Matching across scans: exact `id` match ⇒ same monster (span/HP/evidence refreshed). For span species (Zombie/Mimic/Skeleton), if the fingerprint changed but a same-species finding overlaps ≥ 50% of the old span in the same file, treat as the same monster (id is rewritten to the new fingerprint; `aka` history kept, max 5 entries).
- Renames: from the scan's `diff-tree -M50` output, build `old_path → new_path`; migrate ids before matching (monsters move rooms; event-silent).
- Unmatched old `alive` monsters → slain/fled per §3.5. New findings without a match → `monster_appeared`.

### 6.6 Language comment map

Static table `ext → {line_comment_markers[], block_comment_pairs[], code_like bool}` covering at minimum: Go, JavaScript/TypeScript (+JSX/TSX), Python, Ruby, Rust, Java, Kotlin, C/C++/ObjC (h/c/cc/cpp/m/mm), C#, PHP, Swift, Shell (sh/bash/zsh), PowerShell, SQL, HTML/CSS/SCSS, YAML, TOML, Lua, Perl, Elixir, Haskell, Markdown (`code_like=false`). Unknown extensions: no Zombie detection; other species still apply. Table format + exhaustive list in issue 15.

---

## 7. State & Persistence

### 7.1 Locations & repo identity

- Data root: `${GITQUEST_DATA_DIR}` if set; else `${XDG_DATA_HOME}/gitquest`; else `~/.local/share/gitquest` (all platforms, including macOS — CLI convention, keeps paths predictable and short).
- User config: `${GITQUEST_CONFIG_DIR}` / `${XDG_CONFIG_HOME}/gitquest` / `~/.config/gitquest`.
- Profile dir per repo: `profiles/<repo-id>/` where `repo-id = first16hex(root commit SHA; if the repo has multiple root commits, the lexicographically smallest) + "-" + slug(basename(root))`; `slug` = lowercase, `[a-z0-9-]` only, max 32 chars, empty → `repo`. Root-SHA prefix collision (different repos, same prefix): full-SHA fallback directory; lookup tries short then full.
- Files inside a profile: `state.json` (authoritative), `cache.json` (rebuildable derived data), `lock` (advisory lock), `state.json.bak` (previous good version).
- Permissions: data root `0700`, files `0600` (state can embed paths/commit metadata; treat as private).

### 7.2 `state.json` schema (v1 = schema_version 1)

```jsonc
{
  "schema_version": 1,
  "repo": { "id": "a1b2c3d4e5f60718-gitquest", "root_sha": "…40hex…",
            "last_known_path": "/abs/path", "created_at": 1767000000 },
  "player": {
    "xp": 15230, "level": 11,
    "lifetime": { "commits_scanned": 1042, "insertions": 120345,
                  "deletions": 98123, "slays": 17, "scans": 33 },
    // v2-ready: per-author ledger; v1 writes it but renders only the total.
    "authors": { "<sha256(email_lowercase)>": { "xp": 15230, "commits": 1042 } }
  },
  "last_scan": { "sha": "…", "at": 1767000000, "head_commit_time": 1766990000 },
  "last_scan_health": { "warnings": [], "skipped_files": 0, "budget_hits": [] },
  "monsters": [ {
      "id": "zombie:src/legacy/api.js:9f2c1a0b7d3e4f55",
      "aka": [], "species": "zombie", "name": "Grubmaw the Forgotten",
      "path": "src/legacy/api.js", "span": [120, 154],
      "hp": 68, "level": 4, "status": "alive",
      "evidence": "34-line commented-out code block",
      "first_seen": { "at": 1767000000, "scan_sha": "…" },
      "resolved": null,  // or { "status": "slain", "at": …, "commit": "…" }
      "tame_reason": ""  // optional user note set by `tame --reason` (≤ 200 chars)
  } ],
  "quest_counter": 17,   // monotonic source for quest ids (q-%06d)
  "quests": [ { "id": "q-000017", "monster_id": "…", "issued_at": …,
                "status": "active" } ],   // active|completed|cancelled (last 50 kept)
  "achievements": { "first-blood": { "at": 1767000000 } },
  "events": [ { "at": …, "type": "monster_slain", "data": { … } } ]  // ring, 200
}
```

Numbers are int64 unix seconds. Unknown fields are preserved on rewrite (forward compatibility: decode into typed structs + raw map merge; exact mechanism in issue 09).

### 7.3 Durability rules

- Atomic save: write `state.json.tmp` (fsync) → rename over `state.json`; previous good copy first rotated to `state.json.bak`.
- Advisory lock: `flock`-style exclusive lock on `lock` file held for the whole scan; non-blocking acquire; busy ⇒ exit 1 `another gitquest is exploring this dungeon`.
- Corruption recovery: JSON parse failure ⇒ try `state.json.bak` (with warning); both bad ⇒ exit 4 with recovery instructions (`gitquest reset --hard` recreates; message must state that XP will restart from history replay, monsters/tames are lost).
- `cache.json` corruption: silently rebuilt (it is derived data).
- Schema migration: `schema_version` gate; v1 ships with the framework (ordered migration funcs) but only version 1.

---

## 8. CLI Surface

### 8.1 Command tree & global flags

```
gitquest scan [--fail-on-monsters] [--quiet] [--json]
gitquest status [--json]
gitquest map [--depth N] [--all] [--json]
gitquest monsters [--species S] [--status alive|slain|fled|tamed|all] [--sort hp|age|path] [--json]
gitquest quests [--json]
gitquest achievements [--json]
gitquest tame <monster-id|path[:line]> [--reason TEXT] / gitquest tame --list / --undo <id>
gitquest report [--json] [--fail-on-monsters]
gitquest reset [--hard] [--yes]
gitquest explore
gitquest version
Global flags: --repo PATH (default: cwd discovery), --json (only on the commands
              marked above; others reject it with a usage error),
              --no-color, --theme auto|dark|light|mono, --config PATH, -v/--verbose
```

Notes: `scan` is the ONLY command that mutates game state (plus `tame`/`reset` which mutate targeted parts). `status`/`map`/`monsters`/`quests`/`achievements`/`report`/`explore` read the last scan's state and never re-scan implicitly; if no scan has ever run they print a hint (`Run 'gitquest scan' to generate this dungeon.`, exit 0 for status-like commands, exit 1 for `report --fail-on-monsters`). `NO_COLOR` env var is honored (equivalent to `--no-color`). `--json` output goes to stdout with nothing else; human text goes to stdout, warnings/errors to stderr.

### 8.2 Per-command behavior (summary contracts)

| Command | Reads | Writes | Output essence |
|---|---|---|---|
| `scan` | git, state | state | Event feed (level-ups, appearances, slays, quests) + one-line summary (`Lv 11 · 15,230 XP · 23 monsters (1 boss) · 3 quests`); `--json` emits the §8.3 document plus an `events_this_scan` array |
| `status` | state, git (HEAD `rev-parse` only, for the stale-scan hint) | — | Player card: level, XP bar to next level, monsters alive/slain, active quests, boss line, last 5 events |
| `map` | state | — | Dungeon tree: floors (top-level dirs) with monster counts, guardians, boss marker; `--depth` default 3; branches without monsters collapsed unless `--all` |
| `monsters` | state | — | Table: NAME, SPECIES, LV, HP, ROOM (path:span), AGE, STATUS; default filter alive; `--json` list |
| `quests` | state | — | Up to 3 bounty cards with target, reward preview (`HP × 1.25`), hint (`git log -p -- <path>` suggestion) |
| `achievements` | state | — | 15 slots, locked ones greyed with hint text |
| `tame` | state | state | Marks target monster tamed (id or unique path[:line] resolution; ambiguous ⇒ list candidates, exit 2); `--undo` restores to alive-if-redetected |
| `report` | state | — | Human debt report; `--json` = full machine schema (§8.3) |
| `reset` | state | state | `--hard` deletes profile dir (confirmation unless `--yes`); without `--hard` clears monsters/quests/events but keeps XP/achievements |
| `explore` | state | state (tame only) | Interactive browser (§8.5) |
| `version` | — | — | `gitquest <semver> (<commit>, <date>)` |

### 8.3 JSON report schema (stable contract, `schema_version: 1`)

Top-level: `{schema_version, generated_at, repo{id, root, head_sha}, player{xp, level, xp_into_level, xp_to_next_level, lifetime{…}}, boss{monster_id}|null, monsters[], quests[], achievements[], last_scan{sha, at}, scan_health{warnings[], skipped_files, budget_hits[]}}`. `scan --json` emits the same document with one additional array, `events_this_scan`. Monster objects mirror §7.2 plus `name`, `depth`. Field additions are allowed within schema_version 1; removals/renames require a bump. `report --json` never includes absolute paths except `repo.root`.

### 8.4 Rendering & theming

- All styled output flows through `internal/render`: theme registry (`auto` detects via terminfo/`NO_COLOR`; `dark`/`light`/`mono` forced).
- Sanitizer (security-critical, §10.3): every repo- or git-derived string (paths, evidence, commit subjects, author names, TODO text, branch labels) passes `render.Sanitize` which strips C0/C1 control chars (except `\n`/`\t` where layout requires), all ESC sequences, and truncates to display width. No repo-derived string may reach stdout un-sanitized — enforced by convention + a lint test greping render call sites (issue 28).
- Layout budget: all non-JSON output must degrade to 80-column terminals; wide tables truncate path middles (`src/…/api.js`).

### 8.5 `explore` interactive mode

- bubbletea app with 4 tabs: `[1] Map · [2] Bestiary · [3] Quests · [4] Status`.
- State machine: `Screen ∈ {map, bestiary, quests, status}` × `Mode ∈ {list, detail, confirm}`.
- Keys: `1-4`/`tab`/`shift+tab` switch screens (list mode only); `j/k/↑/↓` move; `enter` detail; `esc` back; in bestiary detail `t` opens tame confirmation (`y`/`n`); `q`/`ctrl+c` quit anywhere (confirm-mode `q` = cancel).
- Data: loaded once from state at startup; `t` (tame) applies the same code path as CLI `tame` (with lock), then updates the in-memory view. No rescanning inside explore (v1).
- Terminal restore on panic (bubbletea's recover + explicit cleanup); minimum size 80×20 with a friendly "window too small" screen.

---

## 9. Configuration

### 9.1 Precedence

`flags > repo .gitquest.toml > user config.toml > built-in defaults`. Exception: security clamps (§9.3) always bound the final value; `tame` data is state, not config.

### 9.2 Repo-local `.gitquest.toml` (UNTRUSTED input)

Read from `<repo-root>/.gitquest.toml` at HEAD (via cat-file, not worktree). Max size 64 KiB. Unknown keys ⇒ warning, not error. No key may cause subprocess execution, network, or filesystem writes; the schema simply has no such capabilities.

```toml
schema = 1
[scan]
exclude = ["legacy/importer/**"]     # doublestar globs, repo-relative
include_only = []                    # if non-empty, candidates must match
use_default_excludes = true
[thresholds]                         # every key clamped, see §9.3
zombie_min_lines = 8
skeleton_min_age_days = 365
ghost_min_age_days = 730
ghost_min_lines = 200
golem_min_lines = 1000
[monsters]
disable = []                         # e.g. ["ghost"]
```

### 9.3 Clamp table (defense against pathological configs)

| Key | Default | Min | Max |
|---|---|---|---|
| `zombie_min_lines` | 8 | 3 | 200 |
| `skeleton_min_age_days` | 365 | 30 | 3650 |
| `ghost_min_age_days` | 730 | 90 | 3650 |
| `ghost_min_lines` | 200 | 50 | 10000 |
| `golem_min_lines` | 1000 | 300 | 50000 |
| `scan.max_files` (user cfg only) | 100000 | 1000 | 500000 |
| `scan.max_total_bytes` (user cfg only) | 512 MiB | 64 MiB | 4 GiB |
| `skeleton_blame_budget` (user cfg only) | 500 | 0 | 5000 |

User `config.toml` adds `[ui] theme = "auto"`, `color = "auto|always|never"`, and may set budget keys; repo config may NOT raise budgets (only defaults/user config can) — repo config raising resource use would let a hostile repo amplify DoS (§10.2).

---

## 10. Security Model

### 10.1 Trust boundaries & threat model

| Boundary | Trusted? | Threats | Mitigations |
|---|---|---|---|
| Target repository content (blobs, paths, commit metadata, `.gitquest.toml`) | NO | ANSI/control-char injection into terminal; path traversal via crafted names; decompression/size bombs; pathological line counts; malicious config | §10.2–§10.5 |
| System `git` binary | YES (system tool) | PATH hijack in odd setups | absolute-path resolution via `exec.LookPath` once, documented assumption |
| User config | Semi (user's own) | foot-guns | clamps §9.3 |
| GitQuest state dir | YES (0700) | tampering by other local users | perms; corruption recovery §7.3 |
| Terminal | — | escape-sequence side effects | central sanitizer §10.3 |
| Network | N/A | — | structurally absent (I-2): no net imports; CI check greps for `net/http` etc. (issue 43) |

Out of scope (documented): defending against a hostile *git binary*, hostile local root, or side channels; scanning repos with actively malicious git hooks is safe because the allowlisted read commands do not trigger hooks.

### 10.2 Untrusted repository rules

- Never execute repo content. No hooks, no scripts, no `git config` reads that change execution (config is read via plumbing output only, and we never run repo-configured aliases/pagers: every invocation passes `--no-pager` and `-c core.pager=cat` is unnecessary given `--no-pager`; aliases don't apply to plumbing argv).
- Repo config cannot raise resource budgets (§9.3) and cannot point excludes outside the repo (globs are matched against repo-relative paths only; absolute or `..`-containing patterns are rejected with a warning).
- Paths from git are validated: must be relative, no `..` segment, no NUL; violating entries are skipped with a warning (they cannot occur from well-formed git output; treat presence as hostile).

### 10.3 Terminal output safety

All repo-derived strings are sanitized (§8.4). Acceptance for every rendering issue includes an ANSI-injection test: a fixture file named `evil-\x1b]0;pwned\x07.go` and a TODO containing `\x1b[2J` must render as escaped/stripped text, never as live control sequences. JSON output relies on `encoding/json` escaping (control chars are `\u`-escaped by the encoder; test asserts this).

### 10.4 Filesystem safety

- Content via `cat-file` only ⇒ no worktree symlink traversal, no reading outside the repo objects, no TOCTOU with a concurrently-changing worktree.
- Symlink blobs (mode 120000) and submodules (160000) are excluded from candidates (§5.2).
- State writes are confined to the profile dir; profile dir name components are derived from SHA-hex + a sanitized slug (§7.1) — no repo-controlled bytes reach the filesystem path except through that slug sanitizer (`[a-z0-9-]`, length-capped).

### 10.5 Resource limits (DoS resistance)

Budgets in §6.2 (file count, total bytes, per-file size, blame count, timeouts) are hard limits with explicit skip-reporting. Decoding uses streaming line splitting with a per-line cap (lines > 64 KiB are truncated for analysis, flagged). The `cat-file --batch` reader enforces declared-size checks before buffering. Memory target: peak RSS ≤ ~512 MiB on the max-budget scan.

### 10.6 Subprocess safety

§5.1: allowlisted argv, no shell, `--` separators, scrubbed environment, timeouts, `--no-optional-locks`. stderr from git is captured, size-capped (64 KiB), sanitized before display in verbose mode.

### 10.7 Supply chain & release security

- Dependencies (target set): `cobra`, `bubbletea`, `lipgloss`, `bubbles`, `doublestar`, `BurntSushi/toml`, `x/term`. Anything beyond requires justification in the PR.
- CI: `go vet`, `golangci-lint`, `govulncheck`, tests with `-race`; Actions pinned by commit SHA; minimal `permissions:` blocks; Dependabot enabled.
- Releases: goreleaser builds with `CGO_ENABLED=0`, `-trimpath`; SHA256SUMS published; version/commit/date injected via `-ldflags`. (Artifact signing: v2.)
- `SECURITY.md` with private reporting via GitHub security advisories.

### 10.8 Privacy

State may contain: absolute repo path, commit SHAs, author-email *hashes* (sha256, lowercase-normalized) — never raw emails; file paths; TODO excerpts (≤ 120 chars each, evidence lines). Documented in README. `report --json` includes no author data in v1.

---

## 11. Errors & Exit Codes

| Code | Meaning | Examples |
|---|---|---|
| 0 | Success | includes "nothing new" scans |
| 1 | Runtime failure | git call failed, lock busy, I/O error |
| 2 | Usage error | unknown flag, ambiguous tame target |
| 3 | Unusable repository | not a repo, empty repo, bare repo, git too old |
| 4 | State corrupted | both state.json and .bak unreadable |
| 5 | Monsters present | only with `--fail-on-monsters` and ≥ 1 living monster |

Error style: one-line human message (themed, sanitized) + optional hint line; `-v` adds the underlying error chain to stderr. Never print raw git stderr unsanitized.

---

## 12. Performance Targets

| Scenario (reference: linux-history-scale excluded; targets for ≤ mid-size repos) | Target |
|---|---|
| First scan, 10k files / 50k commits, warm FS cache | ≤ 30 s |
| Incremental scan, 100 new commits / 300 changed files | ≤ 3 s |
| `status`/`monsters`/`map` (state read only) | ≤ 150 ms |
| Peak RSS during max-budget scan | ≤ 512 MiB |

Benchmarks + a synthetic fixture generator enforce these in issue 42 (soft gates: regression fails CI at +50%).

---

## 13. Testing & Validation Strategy

- Unit: `game` (formulas — golden tables for XP/levels/HP), `detect` (per-species fixture files, false-positive corpora), `render` (sanitizer property tests), `config` (clamps), `state` (atomicity via crash-injection, migrations).
- Integration: fixture repo builder (issue 41) constructs real git repos in temp dirs via scripted `git` commands (deterministic timestamps with `GIT_AUTHOR_DATE`/`GIT_COMMITTER_DATE`), then runs the real binary; golden JSON assertions on `report --json`; scenario suite covers: first scan, incremental slay, fled (threshold change), tame, rename tracking, rewrite recovery, shallow clone, empty repo, non-repo.
- Security tests: ANSI injection fixtures (§10.3), hostile path entries, oversized `.gitquest.toml`, budget exhaustion.
- TUI: bubbletea model unit tests (message-driven, no PTY) + one smoke test with `teatest`.
- CI matrix: {ubuntu-latest, macos-latest} × Go {stable, oldstable}; Windows build-only job.

---

## 14. Release & Distribution

- Versioning: SemVer, `v0.x` during development; `v1.0.0` = ISSUE_PLAN complete.
- goreleaser: darwin/linux (amd64, arm64) archives + checksums; windows amd64 zip built by a separate non-blocking step (never gates a release, per §2.3); Homebrew tap formula in `Saber5656/homebrew-tap` (publication step manual per repo rules — agent prepares config only).
- README: hero GIF (asciinema→gif of scan + status), quickstart, monster field guide table, config reference, privacy note, FAQ ("my code isn't dead!" → `tame`).

---

## 15. Glossary

| Term | Meaning |
|---|---|
| Dungeon | The scanned repository |
| Floor / Room | Top-level directory / file |
| Monster | A detected dead-code finding |
| HP | Debt magnitude; slay XP bonus |
| Slay | Finding resolved by commits touching its file |
| Flee | Finding disappeared without a related commit (no XP) |
| Tame | User-declared false positive (permanent ignore) |
| Boss / Guardian | Max-HP monster of the dungeon / of a floor |
| Bone | One fossilized TODO line inside a Skeleton |
| Adventure log | Commit history converted to XP on first scan |
