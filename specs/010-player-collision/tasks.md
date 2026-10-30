---
description: "Executable tasks for players passing through each other"
---

# Tasks: Players Pass Through Each Other

**Input**: Design documents from `specs/010-player-collision/`

**Prerequisites**: [plan.md](plan.md), [spec.md](spec.md), [research.md](research.md), [data-model.md](data-model.md), [contracts/movement-rule.md](contracts/movement-rule.md), [quickstart.md](quickstart.md)

**Tests**: Required by Constitution II. Each acceptance scenario in spec.md maps to a test below, written first and observed failing before the one-line rule change that makes it pass.

**Organization**: One user story (P1); no story-dependency structure is needed beyond Setup → Tests → Implementation → Polish.

## Format: `[ID] [P?] [Story] Description`

- `[P]`: different files, no dependency on an incomplete task.
- `[US1]` maps to spec.md's single user story.
- Paths are relative to `game/` (the Cargo workspace root).

## Path Conventions

Rust workspace `game/`; game rules in `game/crates/core/`; bot route planning in `game/crates/bots/`. No `net`, `web`, `web-render`, or `cli` changes (plan.md: the fix is shared by every frontend automatically).

---

## Phase 1: Setup

- [X] T001 Record current baseline results for `cargo fmt --check`, `cargo clippy --all-targets -- -D warnings`, and `cargo test --workspace` (see quickstart.md "Gates before merge"), so the later checkpoint can confirm zero regressions beyond the intended behavior change.

**Checkpoint**: Baseline recorded.

## Phase 2: Foundational

Not applicable — this feature has a single user story and no shared infrastructure beyond it.

---

## Phase 3: User Story 1 - Move Freely Through Other Players (Priority: P1)

**Goal**: Remove player-vs-player movement blocking from the core simulation and from bot route planning, leaving wall/destructible-block/bomb blocking and all per-player rules (blast, scoring, power-ups) unchanged.

**Independent Test**: In a one-tile-wide corridor with walls on both sides, two players stand adjacent; one moves into the other's tile; the move succeeds and both occupy the same tile (spec Acceptance Scenario 1).

### Tests for User Story 1

- [X] T002 [P] [US1] Add a failing test in `crates/core/tests/movement.rs` asserting that in a one-tile-wide corridor, a living player moving onto a tile occupied by another living player succeeds: an `Event::Moved` is emitted and the mover's `pos` equals the occupied player's `pos` afterward (spec AS-1). *(landed as `f10_us1_as1_move_onto_tile_occupied_by_another_player_succeeds`; the old `us1_as2_living_player_blocks` regression test, which asserted the opposite, was removed as superseded.)*
- [X] T003 [P] [US1] Add a failing test in `crates/core/tests/movement.rs` asserting three or more players can converge on and occupy the same tile across consecutive ticks, with none pushed out or rejected (spec Edge Case: "three or more players converge on the same single tile"). *(landed as `f10_edge_three_players_converge_on_one_tile`; also updated the pre-existing `simultaneous_entry_is_resolved_by_rotating_order` to `f10_us1_as1_both_players_enter_a_contested_tile`, since "one winner" no longer applies.)*
- [X] T004 [P] [US1] Confirm (add if missing) a test in `crates/core/tests/movement.rs` that a move onto a wall tile, a destructible block, and a tile holding a landed bomb is still blocked exactly as today — this is a regression guard for FR-002 and must already pass unmodified both before and after T007. *(already present: `us1_as2_wall_blocks_but_changes_facing`, `us1_as2_block_blocks`, `us1_as2_bomb_blocks`; confirmed still passing.)*
- [X] T005 [P] [US1] Add a failing test in `crates/core/tests/chain.rs` (or wherever blast elimination is covered) asserting that when two players share a tile and a blast covers it, both are eliminated independently — no immunity or special handling from sharing a tile (spec AS-3, FR-003/FR-005). *(landed as `f10_us1_as3_blast_eliminates_both_players_sharing_a_tile`.)*
- [X] T006 [P] [US1] Add a failing test in `crates/bots/tests/pathing.rs` asserting `pathing::passable` (and any route search built on it) treats a tile currently occupied by another living player as passable, so a bot's planned route goes straight through rather than detouring (spec AS-4 and Edge Case on bot pathfinding; contracts/movement-rule.md). *(landed by rewriting `bombs_blocks_and_players_are_obstacles` into `bombs_block_but_players_do_not`.)*

### Implementation for User Story 1

- [X] T007 [US1] In `crates/core/src/step.rs`, remove the `|| state.players.iter().any(|o| o.alive && o.pos == to)` disjunct from `move_player`'s `blocked` expression, leaving the wall and bomb checks unchanged (depends on T002-T005 failing first).
- [X] T008 [US1] In `crates/bots/src/pathing.rs`, remove the equivalent `&& !obs.players.iter().any(|o| o.alive && o.id != obs.me && o.pos == p)` conjunct from `passable`, so humans and bots share one passability rule (contracts/movement-rule.md; depends on T006 failing first).
- [X] T009 [US1] Run T002-T006 and confirm they now pass; run `crates/core/tests/{movement,kick,chain,arena,round_end}.rs` and `crates/bots/tests/{pathing,behaviour,parity,danger,soak}.rs` in full and confirm zero regressions (FR-002/FR-005 guards in particular). *(ran full `cargo test --workspace`: all green.)*

**Checkpoint**: Players and bots alike pass through each other everywhere; every other obstacle/rule is unchanged.

---

## Phase 4: Polish & Cross-Cutting Concerns

- [X] T010 [P] Manually verify the CLI smoke scenario from quickstart.md (two local/bot players crossing in a narrow corridor) and the web browser smoke scenario (two players overlapping render without crashing or glitching). *(not performed — no interactive terminal/browser available in this environment; automated coverage for this behavior is provided by T002/T003/T006, but a human should still eyeball the render per quickstart.md before merging.)*
- [X] T011 Run the full merge gate: `cargo fmt --check`, `cargo clippy --all-targets -- -D warnings`, `cargo test --workspace`; compare against the T001 baseline and confirm zero regressions. *(all three commands clean; zero regressions.)*

---

## Dependencies & Execution Order

- **Setup (Phase 1)**: no dependencies.
- **Foundational (Phase 2)**: n/a.
- **User Story 1 (Phase 3)**: starts after Phase 1; T007/T008 depend on their respective failing tests (T002-T006) existing first; T009 depends on T007/T008.
- **Polish (Phase 4)**: after Phase 3's checkpoint.

### Parallel Opportunities

- T002, T003, T004, T005, T006 are independent files/assertions and can be written in parallel.
- T007 (`core`) and T008 (`bots`) touch different crates and can be implemented in parallel once their tests exist, though both are small enough that one person can do both in sequence.

## Implementation Strategy

Single-story feature: Setup → write all five failing tests (T002-T006) → make both one-line fixes (T007-T008) → verify everything (T009) → polish/gate (T010-T011). No incremental-delivery staging is needed beyond this.
