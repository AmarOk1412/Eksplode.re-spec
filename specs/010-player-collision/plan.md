# Implementation Plan: Players Pass Through Each Other

**Branch**: `010-player-collision` | **Date**: 2026-10-29 | **Spec**: [spec.md](spec.md)

**Input**: Feature specification from `specs/010-player-collision/spec.md`

## Summary

Remove player-vs-player movement blocking from the core simulation and from bot pathfinding, while
leaving wall/destructible-block/bomb blocking untouched. The entire rule change is one boolean condition
in `core::step::move_player`'s `blocked` check (`crates/core/src/step.rs`) plus the mirrored obstacle
check bots use for route planning (`crates/bots/src/pathing.rs::passable`). No other subsystem (blast
damage, scoring, power-ups, win conditions, bomb-kick/slide) currently depends on player-vs-player
blocking, so no other change is required.

## Technical Context

**Language/Version**: Rust stable, workspace edition 2021, MSRV 1.75; change is confined to `crates/core`
and `crates/bots`.

**Primary Dependencies**: None added; no change to `net`, `web`, `web-render`, or `cli` — all frontends
consume `core`'s simulation output unchanged (Principle III: core/frontend separation), so this is a
single-source-of-truth fix that automatically applies to both the CLI and web clients (Principle VI).

**Storage**: N/A.

**Testing**: `cargo test --workspace`, specifically `crates/core/tests/movement.rs` (and any other
existing test in `kick.rs`/`chain.rs`/`arena.rs`/`round_end.rs` that asserts or relies on player-vs-player
blocking) plus new/updated `crates/bots` pathing tests.

**Target Platform**: Unaffected — this is simulation logic shared by every platform the game already
supports.

**Project Type**: Existing Rust workspace; no new project type.

**Performance Goals**: No measurable change — the `blocked` check drops one `O(players)` condition, a net
simplification; no new allocation or iteration pattern introduced.

**Constraints**: Wall/destructible-block/bomb blocking (`Tile::Floor` check and the bomb-position check in
the same `blocked` expression) MUST remain exactly as-is (FR-002); blast damage, scoring, power-up pickup,
and win-condition logic (which already operate per-player, independent of tile sharing) MUST NOT change
(FR-005); the bomb-kick entry point (`kick::start`, called when `blocked` is true) is unaffected since it
already no-ops unless the blocking obstacle at `to` is specifically a landed, non-sliding bomb.

**Scale/Scope**: Two call sites (`crates/core/src/step.rs::move_player`, `crates/bots/src/pathing.rs::passable`);
no protocol, wire format, or rendering change — overlapping players are already representable in the
existing position model (multiple `Player`/`Participant` entries can already hold equal `pos` values
whenever a tie or edge case currently produces one, e.g. around round start), so no renderer crash/glitch
risk is expected, but the web renderer's handling of two sprites at the same tile is verified in
quickstart.md.

## Constitution Check

*GATE: checked before research and again after Phase 1 design.*

| Principle/Gate | Pre-design | Post-design |
| --- | --- | --- |
| I. Specification-first | PASS: spec.md's single user story precedes this plan | PASS: plan traces directly to FR-001..005 |
| II. Test-first & full coverage | PASS: existing movement tests must be found and updated/added to fail first, then pass | PASS: quickstart.md lists the exact failing-then-passing test additions |
| III. Core/frontend separation | PASS: fix lives entirely in `core`/`bots`, no frontend rule duplication | PASS: no `net`/`web`/`cli` changes needed; frontends inherit the fix automatically |
| IV. Deterministic simulation | PASS: removing a condition from a pure function keeps `step` deterministic | PASS: no RNG, no wall-clock, no new ordering dependency introduced |
| V. Four-player multiplayer with bots | PASS: bots must experience the same passability rule as humans (FR-004) | PASS: `pathing::passable` updated in lockstep with `step::move_player` so bots and humans share one rule, not two divergent copies |
| VI. Dual interface parity | PASS: single shared `core` fix reaches CLI and web identically | PASS: no interface-specific code path exists for this rule today, so none is introduced |
| Library must not panic / `Result` errors | PASS: no new fallible operation introduced | PASS: unchanged — this is a condition removal, not new I/O or parsing |
| Documentation/merge gates | PASS: `cargo fmt`/`clippy -D warnings`/`cargo test --workspace` required as always | PASS: quickstart.md gates section restates this |

No Constitution exceptions.

## Project Structure

### Documentation (this feature)

```text
specs/010-player-collision/
├── spec.md
├── checklists/requirements.md
├── plan.md            # this file
├── research.md        # Phase 0 output
├── data-model.md       # Phase 1 output (trivial: no new entities, documents the removed invariant)
├── contracts/           # Phase 1 output (behavioral contract only; no wire/API change)
│   └── movement-rule.md
└── quickstart.md        # Phase 1 output
```

`tasks.md` is deferred to `/speckit-tasks`.

### Source Code (repository root: `game/`)

```text
crates/core/src/step.rs         # move_player(): drop the player-occupancy clause from `blocked`
crates/core/tests/movement.rs   # update/add tests: moving onto an occupied tile now succeeds;
                                 # wall/block/bomb blocking still rejected; overlap persists until
                                 # one player moves away
crates/core/tests/kick.rs,
crates/core/tests/chain.rs,
crates/core/tests/arena.rs,
crates/core/tests/round_end.rs  # audit for any assertion that happens to rely on player-blocking;
                                 # update if found (none are expected to target this rule directly)

crates/bots/src/pathing.rs      # passable(): drop the other-living-player clause
crates/bots/tests/              # add/update a pathing test: a route through a tile currently
                                 # occupied by another player is considered passable
```

**Structure Decision**: No new files, crates, or modules. This is a two-call-site behavioral fix within
the existing `core` and `bots` crates.

## Complexity Tracking

*No constitution violations requiring justification.*
