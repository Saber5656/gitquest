# GitQuest — v1 Issue Plan

Derived from `docs/DESIGN.md`. GitHub Issues are generated from `docs/issues/NN-*.md`;
this file is the roadmap and dependency map. If GitHub Issues and these docs disagree,
these docs win.

## 1. v1 completion statement

v1 is complete when **all 48 issues below are implemented and validated**. At that
point GitQuest ships as `v1.0.0`: a single-binary Go CLI that scans any git
repository read-only, converts commit history to XP/levels, detects six monster
species via built-in heuristics, tracks slay/fled/tame lifecycles across scans,
offers quests/boss/achievements, renders a map/bestiary/status, supports taming
false positives, emits a stable JSON report with `--fail-on-monsters`, provides a
lightweight interactive `explore` mode, and is released with a hardened offline
security posture (DESIGN §10) and checksummed artifacts. Anything not covered by
these issues is a v2 item (DESIGN §2.4) or a known unknown (§8 below).

## 2. Issue list (recommended execution order)

| # | File | Title | Wave |
|---|---|---|---|
| 01 | `issues/01-bootstrap-project.md` | Bootstrap Go module, project layout, Makefile, lint config | 0 |
| 02 | `issues/02-ci-workflow.md` | CI: lint, test (race), build matrix, govulncheck, SHA-pinned actions | 0 |
| 03 | `issues/03-community-files.md` | LICENSE (MIT), README skeleton, CONTRIBUTING, SECURITY | 0 |
| 04 | `issues/04-git-runner.md` | Hardened read-only git subprocess runner | 1 |
| 05 | `issues/05-repo-discovery-identity.md` | Repo discovery, preflight checks, repo identity, degraded modes | 1 |
| 06 | `issues/06-history-reader.md` | Commit history stream reader (numstat, incremental, rewrite recovery) | 1 |
| 07 | `issues/07-tree-blob-reader.md` | HEAD tree snapshot + blob content reader with classification | 1 |
| 08 | `issues/08-blame-reader.md` | Blame reader (porcelain parser → line ages) | 1 |
| 09 | `issues/09-data-paths-profiles.md` | Data dir resolution, repo profile dirs, permissions | 2 |
| 10 | `issues/10-state-store.md` | State schema v1, atomic store, locking, recovery, migration framework | 2 |
| 11 | `issues/11-derived-cache.md` | Derived cache store (file last-touched index) | 2 |
| 12 | `issues/12-config-defaults-user.md` | Config engine: defaults, user config, clamp table | 2 |
| 13 | `issues/13-repo-config-untrusted.md` | Repo-local `.gitquest.toml` loader (untrusted hardening) | 2 |
| 14 | `issues/14-domain-types-events.md` | Core domain types & event model | 3 |
| 15 | `issues/15-xp-engine.md` | XP engine (commit folding, purge bonus, author ledger) | 3 |
| 16 | `issues/16-level-curve.md` | Level curve & level-up events | 3 |
| 17 | `issues/17-language-comment-map.md` | Language comment map | 3 |
| 18 | `issues/18-default-exclusions.md` | Default exclusions & generated-file detection | 3 |
| 19 | `issues/19-detector-pipeline.md` | Detector interface & scan pipeline orchestrator (budgets, workers) | 3 |
| 20 | `issues/20-zombie-detector.md` | Zombie detector (commented-out code blocks) | 3 |
| 21 | `issues/21-skeleton-detector.md` | Skeleton detector (fossilized TODOs) | 3 |
| 22 | `issues/22-ghost-detector.md` | Ghost detector (abandoned large files) | 3 |
| 23 | `issues/23-slime-detector.md` | Slime detector (backup/leftover artifacts) | 3 |
| 24 | `issues/24-mimic-detector.md` | Mimic detector (committed conflict markers) | 3 |
| 25 | `issues/25-golem-detector.md` | Golem detector (monolithic files) | 3 |
| 26 | `issues/26-reconciliation.md` | Monster identity & reconciliation (slain/fled/tamed, renames) | 3 |
| 27 | `issues/27-boss-naming.md` | Boss/guardian selection & deterministic naming | 3 |
| 28 | `issues/28-quest-engine.md` | Quest engine | 3 |
| 29 | `issues/29-achievement-engine.md` | Achievement engine & v1 achievement set | 3 |
| 30 | `issues/30-cli-skeleton.md` | CLI skeleton: root command, global flags, exit codes, version | 4 |
| 31 | `issues/31-render-sanitizer.md` | Render layer: themes + repo-string sanitizer (ANSI defense) | 4 |
| 32 | `issues/32-scan-command.md` | `scan` command (pipeline wiring, event feed output) | 4 |
| 33 | `issues/33-status-command.md` | `status` command | 4 |
| 34 | `issues/34-map-command.md` | `map` command (dungeon tree) | 4 |
| 35 | `issues/35-monsters-command.md` | `monsters` command (bestiary table) | 4 |
| 36 | `issues/36-quests-command.md` | `quests` command | 4 |
| 37 | `issues/37-achievements-command.md` | `achievements` command | 4 |
| 38 | `issues/38-tame-command.md` | `tame` command (ignore/undo/list) | 4 |
| 39 | `issues/39-report-command.md` | `report` command (JSON schema, `--fail-on-monsters`) | 4 |
| 40 | `issues/40-reset-command.md` | `reset` command | 4 |
| 41 | `issues/41-explore-shell.md` | `explore` TUI shell & navigation state machine | 5 |
| 42 | `issues/42-explore-map-bestiary.md` | `explore` views: map & bestiary (detail + tame confirm) | 5 |
| 43 | `issues/43-explore-quests-status.md` | `explore` views: quests & status | 5 |
| 44 | `issues/44-e2e-fixtures.md` | E2E fixture repo builder & integration scenario suite | 6 |
| 45 | `issues/45-performance-benchmarks.md` | Performance benchmarks & budget enforcement tests | 6 |
| 46 | `issues/46-security-audit.md` | Security audit & injection test suite (cross-cutting) | 6 |
| 47 | `issues/47-release-pipeline.md` | Release pipeline (goreleaser, checksums, version injection) | 6 |
| 48 | `issues/48-user-docs.md` | User documentation (README, field guide, config reference) | 6 |

## 3. Implementation waves

| Wave | Theme | Issues | Parallelizable? |
|---|---|---|---|
| 0 | Foundation | 01–03 | 02/03 after 01 |
| 1 | Git I/O layer | 04–08 | 05–08 after 04 (06/07/08 mutually parallel) |
| 2 | State & config | 09–13 | 09→10→11; 12→13; the two chains parallel |
| 3 | Game core | 14–29 | 14 first; then 15/16 chain, 17/18 parallel, 19 after 17/18; 20–25 parallel after 19; 26–29 after 14 (26 also after 19) |
| 4 | CLI surface | 30–40 | 30/31 first (parallel); 32 is the integration point; 33–40 parallel after 30/31 |
| 5 | Explore TUI | 41–43 | 41 first; 42/43 parallel |
| 6 | Hardening & release | 44–48 | 44/45/46 parallel; 47 anytime after 02; 48 last |

## 4. Dependency table

Format: issue ← hard prerequisites (soft/parallel notes omitted where obvious).

| Issue | Depends on |
|---|---|
| 01 | — |
| 02, 03 | 01 |
| 04 | 01 |
| 05 | 04 |
| 06 | 04, 05 |
| 07 | 04, 05 |
| 08 | 04 |
| 09 | 01, 05 (repo identity) |
| 10 | 09 |
| 11 | 09, 10 (shared atomic-write helper), 06 (history stream shape) |
| 12 | 01 |
| 13 | 12, 07 (blob read of `.gitquest.toml`) |
| 14 | 01 |
| 15 | 14, 06 |
| 16 | 14, 15 |
| 17 | 01 |
| 18 | 12 |
| 19 | 14, 17, 18, 07 |
| 20 | 19, 17 |
| 21 | 19, 08, 11 |
| 22 | 19, 11 |
| 23 | 19 |
| 24 | 19 |
| 25 | 19 (Lang arrives via FileContext) |
| 26 | 14, 19, 06 |
| 27 | 14, 26 |
| 28 | 14, 26 |
| 29 | 14, 26, 15 |
| 30 | 01, 04, 05, 09, 10, 12 (bootstrap wiring uses all of these) |
| 31 | 12, 30 |
| 32 | 06, 07, 10, 11, 15, 16, 19, 26, 27, 28, 29, 30, 31 |
| 33–37 | 10, 30, 31 (33 also 16 and 04/05 for the HEAD hint; 34 also 27; 35 also 26; 36 also 28; 37 also 29) |
| 38 | 10, 26, 30, 31 |
| 39 | 10, 30, 31 (schema mirrors 10/14) |
| 40 | 09, 10, 30, 31 |
| 41 | 10, 30, 31 |
| 42 | 41, 38 (shared tame path), 34/35 semantics |
| 43 | 41, 42 (shared model + jump target), 33, 36, 37 (content parity) |
| 44 | 32, 05 (error-mode scenarios) + at least 33, 35, 38, 39 (grows with commands) |
| 45 | 32, 44 (fixture generator), 19 (budgets under test) |
| 46 | 02, 04, 07, 13, 19, 31, 39, 44 (harness), 45 (RSS measurement) |
| 47 | 01, 02 |
| 48 | all commands (32–43), 44 (golden snippets), 47, 03 (skeleton replaced) |

## 5. Coverage table (DESIGN.md → issues)

| DESIGN section | Covered by |
|---|---|
| §1 Overview, invariants I-1..I-5 | 03, 04, 46, 48 (invariants re-asserted in every issue's Non-goals) |
| §3.1–3.3 Core loop, dungeon, monsters | 14, 26, 32, 34, 35 |
| §3.4 XP & levels | 15, 16 |
| §3.5 Slaying rules | 26 |
| §3.6 Quests | 28, 36 |
| §3.7 Boss & guardians | 27, 34 |
| §3.8 Achievements | 29, 37 |
| §3.9 Naming | 27 |
| §3.10 Events | 14, 32, 33 |
| §4 Architecture, module layout, pipeline | 01, 19, 32 |
| §5.1–5.2 Git runner & exact usage | 04, 05, 06, 07, 08 |
| §5.3 Incremental algorithm & caches | 06, 11 |
| §5.4 History rewrites | 06 |
| §5.5 Degraded modes | 05, 44 |
| §6.1–6.2 Detector interface, pipeline, budgets | 19 |
| §6.3 Default exclusions | 18 |
| §6.4.1–6.4.6 Species specs | 20, 21, 22, 23, 24, 25 |
| §6.5 Identity & reconciliation | 26 |
| §6.6 Language map | 17 |
| §7.1 Paths & identity | 09 |
| §7.2 State schema | 10 |
| §7.3 Durability | 10 |
| §5.3 / §7.1 derived cache (`cache.json`) | 11 |
| §8.1 Command tree, global flags | 30 |
| §8.2 Command contracts | 32–40 |
| §8.3 JSON report schema | 39 |
| §8.4 Rendering & sanitizer | 31 |
| §8.5 Explore mode | 41, 42, 43 |
| §9 Configuration & clamps | 12, 13 |
| §10 Security model | 04 (subprocess), 13 (untrusted config), 19 (budgets), 31 (ANSI), 46 (audit), 02/47 (supply chain) |
| §11 Exit codes | 30 (mapping), all commands (usage) |
| §12 Performance targets | 45 |
| §13 Testing strategy | 44, 45, 46 + per-issue Validation sections |
| §14 Release & distribution | 47, 48 |
| §15 Glossary | 48 (user docs mirror) |

Every DESIGN section that specifies runtime behavior maps to at least one issue;
no v1 behavior exists only in prose.

## 6. Validation strategy (whole product)

1. **Per-issue gates**: every issue defines Acceptance Criteria + a Validation
   recipe (commands to run). CI (issue 02) enforces `go vet`, `golangci-lint`,
   `go test -race ./...`, `govulncheck` on every PR.
2. **Formula goldens**: XP/level/HP formulas are locked by golden-table unit
   tests (15, 16, 20–25) so later refactors cannot silently change game balance.
3. **Scenario E2E** (44): deterministic fixture repos (fixed
   `GIT_AUTHOR_DATE`/`GIT_COMMITTER_DATE`) drive the real binary through:
   first scan → incremental slay → fled (threshold change) → tame → rename →
   history rewrite → shallow clone → empty/non-repo errors. Assertions run on
   `report --json` goldens (sanitized of volatile fields).
4. **Security suite** (46): ANSI-injection fixtures, hostile path entries,
   oversized/malicious repo config, budget exhaustion, network-import CI grep,
   `--no-optional-locks` verification (target repo mtime snapshot unchanged).
5. **Performance gates** (45): synthetic repo benchmarks with soft CI
   thresholds per DESIGN §12.
6. **Release rehearsal** (47): goreleaser snapshot build in CI on every main
   merge; checksums verified.

## 7. Deferred v2 items

External detector plugins (knip/vulture adapters) over the `Detector` boundary;
per-author parties/leaderboards; full TUI game mode; classes/items/streaks;
badge/gist sharing & shields endpoint; i18n message catalog; watch mode/prompt
integration; artifact signing (sigstore); Windows as a supported platform.
(Authoritative list: DESIGN §2.4.)

## 8. Known unknowns (may create additional issues)

| # | Unknown | Trigger for new issue |
|---|---|---|
| U-1 | Heuristic false-positive rate | Fixture corpus results in 44 force threshold changes beyond config defaults |
| U-2 | Blame cost on huge repos | 45 shows Skeleton budget insufficient → persistent line-age cache issue |
| U-3 | Windows terminal quirks | Best-effort job failures that are cheap to fix → small follow-up issues |
| U-4 | Shallow/partial clone edge cases | 44 scenario failures → degrade-path issues |
| U-5 | >100k-file monorepos | User reports post-release → budget/streaming issues |
| U-6 | Git version spread < 2.30 | Probe telemetry impossible (offline) → docs-only unless reports arrive |
| U-7 | `ls-tree --long` / `log` output edge cases (unusual modes, quoted paths) | Parser fuzz findings in 06/07 → parser hardening issues |
