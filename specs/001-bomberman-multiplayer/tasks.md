---

description: "Task list for Bomberman Multiplayer Game"
---

# Tasks: Bomberman Multiplayer Game

**Input**: Design documents from `/specs/001-bomberman-multiplayer/`

**Prerequisites**: plan.md, spec.md, research.md, data-model.md, contracts/, quickstart.md

**Tests**: REQUIRED (constitution principle II, FR-019). In every phase the test tasks come first and MUST be seen failing before the matching implementation tasks.

**Organization**: Grouped by user story in delivery order (FR-020): US1 terminal, US2 bots, US3 network, US4 isometric GUI.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: can run in parallel (different files, no dependency on incomplete tasks)
- **[Story]**: US1..US4 (user story phases only)
- All paths are relative to `game/` (its own git repository) unless prefixed with `specs/`
- Test names for spec scenarios follow `usX_asY_<what>` (X = story, Y = acceptance scenario number)

## Commit Convention (applies to every task group)

Commit when a task or small group of tasks is green, using:

```text
part: short desc (subject <= 80 chars)

long description to explain the feature
on multiple lines
```

`part` is one of `core`, `bots`, `net`, `cli`, `gui`, `build`, `docs`, `test`. Commits are created only when the user asks; each `Commit:` line below is the suggested subject.

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Standalone repository, workspace and quality gates

- [X] T001 Create `game/` and run `git init` inside it (separate repo from the Spec Kit workspace); add `game/.gitignore` ignoring `target/`
- [X] T002 Create workspace manifest `game/Cargo.toml` (resolver 2, `members = ["crates/*"]`, shared `[workspace.package]` edition 2021, `[workspace.lints]` with `unsafe_code = "forbid"`) and `game/rust-toolchain.toml` (stable, components rustfmt + clippy)
- [X] T003 [P] Create `game/.githooks/commit-msg` script that rejects subjects not matching `^(core|bots|net|cli|gui|build|docs|test): .+` or longer than 80 chars (skip merge commits), make it executable, and document `git config core.hooksPath .githooks`
- [X] T004 [P] Add a test for the hook in `game/.githooks/test-commit-msg.sh` covering valid subject, wrong prefix, 81-char subject, missing colon
- [X] T005 [P] Create CI workflow `game/.github/workflows/ci.yml` running `cargo fmt --check`, `cargo clippy --all-targets -- -D warnings`, `cargo test --workspace`
- [X] T006 [P] Create README skeleton `game/README.md` (overview, build, controls placeholder, host/join placeholder, commit convention)
- [X] T007 Configure `core.hooksPath` in the repo, run the gate commands on the empty workspace, and make the first commit. Commit: `build: initialise workspace, lint gates and commit-msg hook`

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: `core` crate skeleton and the small types every story uses

**CRITICAL**: no user story work before this phase is complete

- [X] T008 Create crate `game/crates/core/Cargo.toml` (name `eksplode-re-core`, library only, no dependencies except optional `serde` behind feature `serde`) and `game/crates/core/src/lib.rs` with `#![forbid(unsafe_code)]`, module declarations and doc comments
- [X] T009 [P] Write failing unit tests for `Rng` in `game/crates/core/src/rng.rs` (same seed gives same sequence, different seeds differ, `next_below(n) < n`, `chance(p)` extremes 0.0 and 1.0)
- [X] T010 Implement SplitMix64 `Rng` in `game/crates/core/src/rng.rs` (seeded constructor, `next_u64`, `next_below`, `chance`) to pass T009
- [X] T011 [P] Implement `Dir` (Up/Down/Left/Right, `delta()`), `PlayerId` (0..=3 newtype, checked constructor returning `CoreError`), `Color` (Red, Blue, Green, Yellow) and `PlayerInput { mv: Option<Dir>, bomb: bool }` with `Default` in `game/crates/core/src/action.rs`, with unit tests
- [X] T012 [P] Implement `CoreError` (typed errors with `Display` and `std::error::Error`) in `game/crates/core/src/error.rs`
- [X] T013 [P] Implement `Config` with the defaults from data-model.md (15x13, tick_hz 20, fuse_ticks 60, flame_ticks 10, round_limit_ticks 3600, base_move_cooldown 4, min_move_cooldown 2, start_bombs 1, start_range 2, max_bombs 6, max_range 8, block_density 0.7, powerup_chance 0.3) and `Config::validate()` (odd width and height >= 7, caps >= starts) in `game/crates/core/src/config.rs`, with unit tests
- [X] T014 Run gates (fmt, clippy, test) for `core`. Commit: `core: add rng, config and basic action types`

**Checkpoint**: foundation ready

---

## Phase 3: User Story 1 - Play a local match in the terminal (Priority: P1) MVP

**Goal**: complete Bomberman rules in `core` and a basic ratatui terminal game for 1-4 local humans (FR-001 to FR-007, FR-010 local, FR-017, FR-018 partial)

**Independent Test**: `cargo test -p eksplode-re-core -p eksplode-re-cli`, then `cargo run -p eksplode-re-cli -- --players 1 --bots 0 --seed 42` and play one round (quickstart Stage 1)

### Tests for User Story 1 (write first, must fail) 

- [X] T015 [P] [US1] Arena generation tests in `game/crates/core/tests/arena.rs`: border and (even,even) tiles are `Wall`; four corner start areas (start tile + 2 adjacent tiles) are `Floor`; same `(Config, seed)` gives identical arena; different seeds give different block layouts; hidden power-up map is deterministic (FR-001, FR-017) - `us1_as1_*`
- [X] T016 [P] [US1] Movement tests in `game/crates/core/tests/movement.rs`: move into floor succeeds; wall, block, bomb and living player block movement; cooldown prevents consecutive moves; simultaneous entry into the same tile resolved by `(tick + index) % 4`; dead players cannot move (FR-002) - `us1_as2_*`
- [X] T017 [P] [US1] Bomb tests in `game/crates/core/tests/bombs.rs`: fuse of `fuse_ticks`, cross blast to `range`, walls stop blasts, first block destroyed and stops the ray, bomb cap respected, no second bomb on the same tile, owner can step off own bomb then it blocks (FR-003) - `us1_as3_*`
- [X] T018 [P] [US1] Chain reaction and elimination tests in `game/crates/core/tests/chain.rs`: bomb inside blast detonates in the same tick, multi-bomb chains settle, player on a flame tile is eliminated, blast through owner eliminates owner (FR-004, FR-005) - `us1_as4_*`, `us1_as5_*`
- [X] T019 [P] [US1] Power-up tests in `game/crates/core/tests/powerups.rs`: destroyed block may reveal power-up per hidden map, flame destroys visible power-up, collecting ExtraBomb / Range / Speed changes stats within `max_bombs`, `max_range` and `min_move_cooldown` caps (FR-006) - `us1_as6_*`
- [X] T020 [P] [US1] Round-end tests in `game/crates/core/tests/round_end.rs`: last survivor is `Winner`, simultaneous elimination is `Draw`, `round_limit_ticks` gives `Draw`, `step` after end is a no-op (FR-007) - `us1_as7_*`
- [X] T021 [P] [US1] Determinism and robustness tests in `game/crates/core/tests/determinism.rs`: same seed + same input log gives identical final state and event list over 50 seeds; random inputs for 3600 ticks never panic and preserve invariants (no player in wall, `bombs_active <= bombs_max`) (FR-015, FR-017)
- [X] T022 [P] [US1] CLI view tests using ratatui `TestBackend` in `game/crates/cli/tests/view.rs`: arena glyphs from contracts/cli-interface.md (`##`, `[]`, `()`, `**`, `+B`, `+R`, `+S`, `P1..P4`), status line per player, "terminal too small" screen below 40x20
- [X] T023 [P] [US1] CLI input mapping tests in `game/crates/cli/tests/input.rs`: key sets P1 arrows+Space, P2 WASD+E, P3 IJKL+O, P4 numpad/TFGH+Y; `q`, `r`, `p`; last direction pressed wins per tick; unknown keys ignored
- [X] T024 [P] [US1] CLI argument parsing tests in `game/crates/cli/tests/args.rs`: defaults (1 player), `--players`, `--bots`, `--seed`, `players + bots > 4` is usage error with exit code 2, `--help`

### Implementation for User Story 1

- [X] T025 [P] [US1] Implement `Tile`, `Arena` and generation `Arena::generate(&Config, &mut Rng)` with border/fixed walls, clear start areas and hidden power-up map in `game/crates/core/src/arena.rs`
- [X] T026 [P] [US1] Implement `Participant`, `Bomb`, `Flame`, `PowerUp`/`PowerUpKind`, `Outcome`, `Phase`, `Event` per data-model.md in `game/crates/core/src/state.rs`
- [X] T027 [US1] Implement `GameState::new(config, seed, active_slots)` (spawns in corners, unused slots not alive) in `game/crates/core/src/state.rs` (depends on T025, T026)
- [X] T028 [US1] Implement movement phase of `step` (rotating order, cooldowns, blocking rules, power-up pickup) in `game/crates/core/src/step.rs` (depends on T027)
- [X] T029 [US1] Implement bomb placement, fuse ticking, detonation with chain-reaction fixed point, flame creation, block destruction and power-up reveal in `game/crates/core/src/bomb.rs` and wire into `step` (depends on T028)
- [X] T030 [US1] Implement elimination, flame expiry, round-end detection (`Winner`/`Draw`/limit) and event emission in `game/crates/core/src/step.rs` (depends on T029)
- [X] T031 [P] [US1] Implement `Observation` (public view: no RNG, no hidden power-ups) and `GameState::observe(PlayerId)` in `game/crates/core/src/observation.rs`
- [X] T032 [US1] Run T015-T021 and make all pass; run gates. Commit: `core: implement arena, movement, bombs, chains and round end` with a body listing the rule coverage
- [X] T033 [US1] Create crate `game/crates/cli/Cargo.toml` (name `eksplode-re-cli`, bin `eksplode-re-cli`, deps `ratatui`, `crossterm`, `eksplode-re-core`) and `game/crates/cli/src/main.rs`
- [X] T034 [P] [US1] Implement CLI argument parsing (`--players`, `--bots`, `--seed`, `--help`, `--version`) with usage errors and exit codes in `game/crates/cli/src/args.rs` (T024)
- [X] T035 [P] [US1] Implement key-to-`PlayerInput` mapping for 4 local key sets, `q`/`r`/`p` commands and per-tick input collection in `game/crates/cli/src/input.rs` (T023)
- [X] T036 [P] [US1] Implement rendering to ratatui `Frame` (arena, entities, per-player status, result screen, too-small screen) in `game/crates/cli/src/view.rs` (T022)
- [X] T037 [US1] Implement local game loop at a fixed 20 Hz (terminal setup/restore guard that restores terminal on panic or error, poll input, `step`, draw, results screen, rematch) in `game/crates/cli/src/app.rs`
- [X] T038 [US1] Wire `main.rs` to args and app; return typed errors, no `unwrap` in non-test code (constitution technology rules)
- [X] T039 [US1] Fill the README controls and run sections for the CLI (FR-018) in `game/README.md`
- [X] T040 [US1] Run quickstart Stage 1 manually and gates. Commit: `cli: add terminal game for local players` and `docs: document CLI controls`

**Checkpoint**: US1 complete; terminal game fully playable and tested

---

## Phase 4: User Story 2 - Play against computer-controlled bots (Priority: P2)

**Goal**: bots using only `Observation`, lobby slot selection for bots, bot-vs-bot soak test (FR-008 partial, FR-009)

**Independent Test**: `cargo test -p eksplode-re-bots`, `cargo run -p eksplode-re-cli -- --players 1 --bots 3 --seed 42`

### Tests for User Story 2 (write first, must fail)

- [X] T041 [P] [US2] Danger map tests in `game/crates/bots/tests/danger.rs`: tiles in blast range of pending bombs are dangerous with time-to-explosion, chain reactions extend danger, walls and blocks shield, flames are dangerous
- [X] T042 [P] [US2] Pathfinding tests in `game/crates/bots/tests/pathing.rs`: BFS finds nearest safe tile, respects bombs/blocks/players as obstacles, path must arrive before detonation, returns `None` when trapped
- [X] T043 [P] [US2] Bot behaviour scenario tests in `game/crates/bots/tests/behaviour.rs`: `us2_as2_flees_blast_when_safe_path_exists`, `us2_as3_does_not_place_bomb_without_escape`, bot places bomb next to a block, bot places bomb near adjacent opponent when escape exists, bot picks up adjacent power-up when safe
- [X] T044 [P] [US2] Information-parity test in `game/crates/bots/tests/parity.rs`: `us2_as5_*` bot compiles against `Observation` only (no `GameState` import) and gives identical decisions when hidden power-ups/RNG differ
- [X] T045 [P] [US2] Bot-only soak test in `game/crates/bots/tests/soak.rs`: `us2_as4_*` 100 four-bot matches with seeds 0..100 all end with `Winner` or `Draw` within `round_limit_ticks`, no panics (SC-003); bot escape rate in scripted scenarios >= 95% (SC-004)
- [X] T046 [P] [US2] Lobby slot model tests in `game/crates/cli/tests/lobby.rs`: `us2_as1_*` assigning a slot to a bot, total players <= 4, removing a bot frees the slot

### Implementation for User Story 2

- [X] T047 [US2] Create crate `game/crates/bots/Cargo.toml` (name `eksplode-re-bots`, deps `eksplode-re-core`) and `game/crates/bots/src/lib.rs` defining `trait Bot { fn decide(&mut self, obs: &Observation) -> PlayerInput }`
- [X] T048 [P] [US2] Implement danger map from `Observation` in `game/crates/bots/src/danger.rs`
- [X] T049 [P] [US2] Implement BFS pathing with arrival-time constraints in `game/crates/bots/src/pathing.rs`
- [X] T050 [US2] Implement `DefaultBot` (priority: flee, collect, attack, break blocks, idle; bomb only with escape; small deterministic jitter from an injected seed) in `game/crates/bots/src/default_bot.rs` (depends on T048, T049)
- [X] T051 [US2] Implement shared lobby slot model (`Controller::{Local, Remote, Bot, Empty}`, `Slot`) in `game/crates/core/src/lobby.rs`, re-exported for frontends, to satisfy T046
- [X] T052 [US2] Integrate bots into the CLI game loop and `--bots` option; add lobby screen with add/remove bot and start in `game/crates/cli/src/lobby.rs` and `game/crates/cli/src/app.rs`
- [X] T053 [US2] Run T041-T046, quickstart Stage 2 and gates. Commits: `bots: add danger map, pathing and default bot`, `cli: add lobby and local bot slots`
- [X] T054 [US2] Update `game/README.md` with bot usage and the soak test command

**Checkpoint**: US1 and US2 complete; single-player games against bots work

---

## Phase 5: User Story 3 - Play with up to 4 players over a network (Priority: P3)

**Goal**: authoritative host plus join clients over TCP/JSON per contracts/network-protocol.md (FR-008, FR-013 to FR-016)

**Independent Test**: `cargo test -p eksplode-re-net`, then host in one terminal and join from others (quickstart Stage 3)

### Tests for User Story 3 (write first, must fail)

- [X] T055 [P] [US3] Codec tests in `game/crates/net/tests/codec.rs`: every message type round-trips; line > 64 KiB rejected; unknown `type`, bad JSON, bad UTF-8 and wrong field types give typed errors, never panic; names must be 1-16 chars
- [X] T056 [P] [US3] Lobby/protocol state tests in `game/crates/net/tests/lobby.rs`: first message must be `Join`; version mismatch gives `Rejected{BadVersion}`; 5th join gives `Rejected{Full}`; join during play gives `Rejected{InMatch}`; ready/start flow
- [X] T057 [P] [US3] Loopback integration tests in `game/crates/net/tests/loopback.rs`: `us3_as1_join_assigns_free_slot`, `us3_as2_all_see_same_start`, `us3_as3_all_clients_agree_on_state_and_outcome`, `us3_as4_disconnect_replaced_by_bot_and_match_continues`, `us3_as5_malformed_and_foreign_slot_inputs_rejected_without_effect`, `us3_as6_fifth_join_refused`; host leaving ends match with `HostLeft`; 10 errors close a connection; 10 s silence counts as disconnect
- [X] T058 [P] [US3] Latency/consistency test in `game/crates/net/tests/consistency.rs`: with 4 loopback clients, 95% of inputs are reflected in a snapshot within 200 ms and final `Ended` outcomes are identical (SC-006, SC-007)

### Implementation for User Story 3

- [X] T059 [US3] Create crate `game/crates/net/Cargo.toml` (name `eksplode-re-net`, deps `eksplode-re-core` with `serde` feature, `eksplode-re-bots`, `serde`, `serde_json`) and `game/crates/net/src/lib.rs`
- [X] T060 [P] [US3] Enable `serde` derives on `core` public types (`Config`, `Dir`, `PlayerInput`, `Event`, `Observation`, `Outcome`, `Slot`) behind feature `serde` in `game/crates/core/src/`
- [X] T061 [P] [US3] Implement protocol messages and `PROTOCOL_VERSION = 1` in `game/crates/net/src/protocol.rs`
- [X] T062 [P] [US3] Implement newline-delimited JSON codec with 64 KiB limit and typed `NetError` in `game/crates/net/src/codec.rs` (T055)
- [X] T063 [US3] Implement host: accept loop, per-connection reader threads feeding an `mpsc` channel, join validation, lobby broadcast, start with seed, 20 Hz authoritative tick loop, per-tick `Snapshot` broadcast, input validation and error counting in `game/crates/net/src/host.rs` (depends on T061, T062)
- [X] T064 [US3] Implement bot takeover on disconnect/timeout and `PlayerLeft` / `Ended` / host-left handling in `game/crates/net/src/host.rs`
- [X] T065 [US3] Implement client: connect, join, ready, send inputs, receive snapshots into a channel-based API usable by frontends in `game/crates/net/src/client.rs`
- [X] T066 [US3] Make loopback tests deterministic and fast (ephemeral port 0, injectable tick duration) in `game/crates/net/src/host.rs`
- [X] T067 [US3] Add `host` and `join` subcommands, remote lobby screen, and remote game rendering from `Snapshot` in `game/crates/cli/src/{args,lobby,app}.rs`; add CLI tests for `host --port --bots --name` and `join ADDR` parsing in `game/crates/cli/tests/args.rs`
- [X] T068 [US3] Run T055-T058, quickstart Stage 3 manually with 2 terminals and gates. Commits: `net: add protocol, codec and authoritative host`, `net: add client and bot takeover on disconnect`, `cli: add host and join modes`
- [X] T069 [US3] Update `game/README.md` with host/join instructions, default port 7878 and the no-encryption note

**Checkpoint**: US1-US3 complete; 4-player remote matches work in the terminal

---

## Phase 6: User Story 4 - Isometric graphical version (Priority: P4)

**Goal**: macroquad isometric client with cubic coloured players, reusing lobby, bots and network (FR-011, FR-012, SC-008, SC-009)

**Independent Test**: `cargo test -p eksplode_re-gui`, `cargo run -p eksplode_re-gui -- --players 1 --bots 3`

### Tests for User Story 4 (write first, must fail)

- [X] T070 [P] [US4] Projection tests in `game/crates/gui/tests/iso.rs`: `sx = (x - y) * TW/2`, `sy = (x + y) * TH/2 - z`; inverse maps back to tiles; the arena bounding box is centred in any window size (replaced by 002)
- [X] T071 [P] [US4] Depth-order tests in `game/crates/gui/tests/depth.rs`: `us4_as3_*` draw list sorted by `(x + y, z)`; a player behind a block is drawn before it; ties are stable (replaced by 002)
- [X] T072 [P] [US4] Cube geometry/colour tests in `game/crates/gui/tests/cube.rs`: `us4_as2_*` top/left/right faces at brightness 1.0/0.8/0.6; the four player colours are pairwise distinct (min colour distance threshold); number badge present (replaced by 002)
- [X] T073 [P] [US4] Parity test in `game/crates/gui/tests/parity.rs`: `us4_as4_*` replaying the same seed and input log produces identical outcomes and identical view-model entities as the CLI view model (replaced by 002)
- [X] T074 [P] [US4] GUI input mapping tests in `game/crates/gui/tests/input.rs`: `us4_as5_*` same key sets as the CLI contract; Esc returns to menu (replaced by 002)
- [X] T075 [P] [US4] Interpolation tests in `game/crates/gui/tests/interp.rs`: position lerp between ticks stays within tile bounds and ends exactly at the target (replaced by 002)

### Implementation for User Story 4

- [X] T076 [US4] Create crate `game/crates/gui/Cargo.toml` (name `eksplode_re-gui`, bin, deps `macroquad`, `eksplode-re-core`, `eksplode-re-bots`, `eksplode-re-net`) and `game/crates/gui/src/main.rs` with `[lib]` for testable modules (replaced by 002)
- [X] T077 [P] [US4] Extract shared client view-model (entities per tile, status, outcome) and key mapping into `game/crates/core/src/view.rs` or a small `eksplode-re-client` crate used by both frontends, adjusting the CLI to use it (keeps parity, constitution III and VI) (replaced by 002)
- [X] T078 [P] [US4] Implement isometric projection and depth-sorted draw list in `game/crates/gui/src/iso.rs` (T070, T071) (replaced by 002)
- [X] T079 [P] [US4] Implement cube face geometry, shading, palette and number badge in `game/crates/gui/src/cube.rs` (T072) (replaced by 002)
- [X] T080 [P] [US4] Implement interpolation in `game/crates/gui/src/interp.rs` (T075) (replaced by 002)
- [X] T081 [P] [US4] Implement keyboard mapping in `game/crates/gui/src/input.rs` (T074) (replaced by 002)
- [X] T082 [US4] Implement rendering of floor, walls, blocks, bombs (pulsing), flames, power-ups and cubes in `game/crates/gui/src/render.rs` (replaced by 002)
- [X] T083 [US4] Implement app: menu/lobby (bots, host, join), local loop at 20 Hz sim with 60 fps rendering, results screen with rematch in `game/crates/gui/src/app.rs` (replaced by 002)
- [X] T084 [US4] Support remote play in the GUI using `eksplode-re-net` client and host with the same options as the CLI (replaced by 002)
- [X] T085 [US4] Add a headless smoke run (`--smoke-frames N` renders N frames then exits with code 0) or document why it is skipped in CI, plus a render timing check for SC-009 in `game/crates/gui/src/main.rs` (replaced by 002)
- [X] T086 [US4] Run T070-T075, quickstart Stage 4 manually (visual check: four distinct cubes, bombs/flames/blocks distinguishable) and gates. Commits: `gui: add isometric projection, depth order and cubes`, `gui: add game app, lobby and remote play` (replaced by 002)
- [X] T087 [US4] Update `game/README.md` with GUI controls, screenshots and run commands (replaced by 002)

**Checkpoint**: all four stories complete

---

## Phase 7: Polish & Cross-Cutting Concerns

- [ ] T088 [P] Add `game/scripts/check-scenarios.sh` verifying that every spec acceptance scenario id `usX_asY` (US1 AS1-7, US2 AS1-5, US3 AS1-6, US4 AS1-5) appears in a test name; run it in CI
- [ ] T089 [P] Audit non-test code for `unwrap`/`expect`/`panic!`/indexing on external data in `game/crates/` and replace with typed errors (constitution technology rules)
- [ ] T090 [P] Add doc comments to all public `core` items and run `cargo doc --no-deps` without warnings
- [ ] T091 Edge-case tests for the spec list in `game/crates/core/tests/edge_cases.rs` (bomb on bomb tile, power-up destroyed by blast, fewer than 4 players start, round limit) and `game/crates/net/tests/edge_cases.rs` (slow connection, host leaves)
- [ ] T092 Review success criteria SC-001 to SC-009 and record measured results (soak, latency, escape rate) in `game/README.md`
- [ ] T093 Run full quickstart.md validation end to end and all gates; fix gaps
- [ ] T094 Review commit history in `game/` against the commit convention (`git log --format=%s | awk 'length>80'` is empty; all subjects match the hook). Commit: `docs: finalise README and success criteria results`

---

## Dependencies & Execution Order

### Phase Dependencies
- **Setup (1)** then **Foundational (2)** block everything.
- **US1 (3)** depends on Phase 2. **US2 (4)** depends on US1 (`Observation`, `step`, CLI loop). **US3 (5)** depends on US1 and US2 (bot takeover, lobby). **US4 (6)** depends on US1-US3 for full feature parity, but its pure modules (T078-T081) can start after US1.
- **Polish (7)** after all desired stories.

### Within a story
- Tests first and failing, then types, then logic, then frontends, then README and commit.
- In US1: T025/T026 before T027; T027 before T028 to T030 (sequential, same `step`); T031 parallel; frontend tasks T034-T036 parallel after T033.

### Parallel Opportunities
- Setup: T003, T004, T005, T006
- Foundational: T009, T011, T012, T013
- US1 tests T015-T024 all in parallel; T025/T026/T031 in parallel; T034/T035/T036 in parallel
- US2 tests T041-T046 in parallel; T048/T049 in parallel
- US3 tests T055-T058 in parallel; T060/T061/T062 in parallel
- US4 tests T070-T075 in parallel; T078-T081 in parallel

### Parallel Example: User Story 1
```text
T015 arena tests | T016 movement tests | T017 bomb tests | T018 chain tests | T019 powerups tests
T020 round end   | T021 determinism    | T022 CLI view   | T023 CLI input  | T024 CLI args
then T025 arena + T026 state + T031 observation, then T027 -> T028 -> T029 -> T030
```

## Implementation Strategy

### MVP First (US1 only)
1. Phases 1-2, then Phase 3; stop and validate with quickstart Stage 1 (terminal game playable).
2. This is the first demonstrable product and the base of all other stages.

### Incremental Delivery
1. US1 terminal game, then US2 bots (solo play), then US3 remote play, then US4 isometric GUI.
2. Each stage ends with gates green, README updated and commits in the required style.

### Notes
- Spec ids `FR-xxx`/`SC-xxx` are referenced in tasks for traceability.
- Never skip the red step: run the new tests and see them fail before implementing.
- Commit only when asked; keep each commit one logical change.
