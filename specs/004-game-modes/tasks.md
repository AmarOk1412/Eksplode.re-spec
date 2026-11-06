# Tasks: Game Modes and Room Settings

**Input**: Design documents from `/specs/004-game-modes/`

**Prerequisites**: plan.md, spec.md, research.md (R1-R12), data-model.md, contracts/ (room-and-protocol, modes-and-rules, cli-and-page, render-and-bots), quickstart.md

**Tests**: Required by the constitution (principle II): every task lands with its tests, written first and seen failing (red), then green. Acceptance scenarios of this feature are named `f4_us<N>_as<M>_*` in test names (the prefix `f4_` keeps them apart from the scenarios of features 002 and 003); scenarios that need eyes or ears are recorded in `specs/004-game-modes/checklists/manual-004.md`.

**Organization**: Tasks are grouped by user story. The phases follow the order requested for delivery (room creation menu, end pattern, other settings, new modes), so the board size and time part of User Story 1 (AS6) is a later phase (Phase 5) that depends on Phases 3 and 4.

## Format: `- [ ] T### [P?] [Story?] Description with file path`

- **[P]**: can run in parallel (different files, no dependency on an unfinished task)
- **[Story]**: US1 room creation and settings, US2 end pattern, US3 Team, US4 Chaser
- Paths are relative to `game/` unless they start with `specs/`
- Commit style: `part: short desc` (at most 80 characters) plus a body; parts `core`, `bots`, `net`, `cli`, `web`, `render`, `build`, `docs`, `test`
- Modes become selectable only when their rules exist: `RoomSettings::sanitize` maps a mode that is not yet playable to Normal (the `PLAYABLE` list in `crates/core/src/room.rs` starts with Normal; Phase 6 adds Team, Phase 7 adds Chaser)

---

## Phase 1: Setup

**Purpose**: scaffolding shared by all stories

- [X] T001 Extend `scripts/check-scenarios.sh` with the 004 ids `f4_us1_as1`..`f4_us1_as8`, `f4_us2_as1`..`f4_us2_as7`, `f4_us3_as1`..`f4_us3_as6`, `f4_us4_as1`..`f4_us4_as9` (new variable `MANUAL_004` defaulting to `../specs/004-game-modes/checklists/manual-004.md`), each satisfied by a test name in `crates/*/tests`, `crates/web/js-tests` or a line in the manual checklist; it must fail while ids are missing (red now, green at the end)
- [X] T002 [P] Create `specs/004-game-modes/checklists/manual-004.md` with empty sections for the checks that need eyes or ears: creation menu and lobby on a phone-sized window (US1 AS1-AS3), the three board sizes on the page and in a terminal (US1 AS6), the announcement, landing and warning and impact sounds (US2 AS6), team colours and the result message (US3 AS5), chaser mark, hearts, protection blink, heart sound and result message (US4 AS9), the dev panel note for Chaser
- [X] T003 [P] Add the 004 gates to `docs/DEVELOPMENT.md` (no new command; note the `f4_` naming rule, the manual checklist, `node --test crates/web/js-tests` and the `--no-setup` option used by `scripts/pty-enter-check.py`)

---

## Phase 2:ca Foundational (blocking prerequisites)

**Purpose**: the core types every story uses, added without changing any behaviour

**CRITICAL**: no user story work starts before this phase is complete

- [X] T004 Before changing any code, record golden values of the current behaviour in `crates/core/tests/determinism.rs`: for seeds 1, 7 and 42 the tile layout of `GameState::new(Config::default(), seed, [true; 4])` as text and the event list of 300 ticks with scripted inputs; test `f4_sc003_default_settings_play_as_before` compares against them (green now, must stay green after T006 and T007)
- [X] T005 [P] Write failing tests in `crates/core/tests/room.rs` for `RoomSettings`: defaults are `name` "", `mode` `Normal`, `size` `Small`, `minutes` 3, `friendly_fire` true; `sanitize` trims the name, removes control characters, cuts it to `at most 24 chars`, maps `minutes` not in `2, 3, 5, 10` to 3 and a mode that is not in `PLAYABLE` to `Normal`; `to_config(&Config::default())` gives 15 x 13 and 3 minutes for the defaults, 19 x 15 for `Medium`, 25 x 19 for `Large` (all odd), `round_limit_ticks` = `minutes * 60 * tick_hz` except that 3 minutes keeps the base value; `participants_needed` is `(1, 4)` for Normal and `(4, 4)` for Team and Chaser; `Config::validate` rejects `closing_lead_ticks == 0`, `chaser_hearts == 0`, `chaser_pillar_mod == 0`, `bonus_every_ticks == 0`, densities outside 0..1 and `chaser_bombs == 0`; closing_ticks == 0 or not shorter than round_limit_ticks turns the pattern off; serde round trip and a JSON without the new fields still deserializes to the old behaviour
- [X] T006 Implement `crates/core/src/room.rs` (new, exported from `crates/core/src/lib.rs`): `Mode { Normal, Team, Chaser }`, `BoardSize { Small, Medium, Large }`, `RoomSettings` with the fields, defaults and rules of T005 (`sanitize`, `to_config`, `participants_needed`, `label`, `PLAYABLE`), doc comments on every public item; extend `crates/core/src/config.rs` with `mode: Mode` (`Normal`), `friendly_fire: bool` (`true`), `closing_ticks: u32` (`60 * tick_hz`), `closing_lead_ticks: u32` (20), `chaser_bombs: u8` (5), `chaser_hearts: u8` (3), `shield_ticks: u32` (40), `chaser_pillar_mod: u8` (3), `chaser_block_density: f32` (0.12), `bonus_every_ticks: u32` (100), `bonus_cap: u8` (6), all `#[serde(default)]`, and the `validate` rules of T005
- [X] T007 Add the inert state fields of `contracts/modes-and-rules.md` and `data-model.md`: in `crates/core/src/state.rs` `Participant` gets `team: Option<u8>` (`None`), `role: Role` (`Normal`; `Role { Normal, Chaser, Runner }`), `hearts: u8` (1), `shield_left: u32` (0); `Flame` gets `owner: PlayerId`; `GameState` gets `closing: Closing` (empty, inactive; `Closing { order: Vec<Pos>, next: usize, elapsed: u32, falling: Vec<Falling> }`, `Falling { pos: Pos, ticks_left: u32 }`) and `bonus_in: u32`; `Outcome` gains `TeamWin(u8)`, `ChaserWins`, `RunnersWin`; `Event` gains `PatternStarted`, `BlockWarned { pos }`, `BlockLanded { pos }`, `HeartLost { player, hearts_left }`, `RoleAssigned { player, role }`; `crates/core/src/observation.rs` gets `falling: Vec<Falling>` and the new participant fields (all `#[serde(default)]` where serialized); fix every non-exhaustive match and constructor in `crates/bots`, `crates/net`, `crates/cli`, `crates/web-render` so the workspace builds with the new variants ignored; `cargo test --workspace` stays green including T004
- [X] T008 [P] Failing test then implementation for the flame owner in `crates/core/tests/chain.rs` and `crates/core/src/bomb.rs`: a flame made by a bomb has `owner` = the owner of that bomb; in a chain reaction every flame has the owner of the bomb that exploded it, not of the bomb that started the chain; two flames on one square keep their own owners

**Checkpoint**: core types exist, behaviour unchanged, gates green

---

## Phase 3: User Story 1 - Create and configure a room (Priority: P1) MVP

**Goal**: a room creation menu (page and terminal), settings shown to everybody in the lobby, owner-only changes before a match, kept for the rematch, room closes when the last person leaves. This phase delivers name and mode and the plumbing for size, time and friendly fire; Normal is the only playable mode and the sizes and times are applied in Phase 5.

**Independent Test**: start a server, open the page as the first person, see the creation menu, pick a name, confirm, see the lobby; a second tab sees the same settings and cannot change them; a terminal client sees them too; the owner's change reaches both at once; a change during a match is refused.

### Tests for User Story 1 (write first, must fail)

- [X] T009 [P] [US1] Lenient decoding tests in `crates/net/tests/room.rs`: `RoomSettings::from_wire(&serde_json::Value)` turns a non-object into the defaults, ignores unknown fields, replaces a missing or wrongly typed field by its default, maps unknown `mode` and `size` strings and `minutes` not in `2, 3, 5, 10` to their defaults (3 for `minutes`), removes control characters, trims and cuts the name to `at most 24 chars`, never fails; wire names are `"normal"`, `"team"`, `"chaser"` and `"small"`, `"medium"`, `"large"`; JSON round trip
- [X] T010 [P] [US1] Host tests in `crates/net/tests/room_host.rs`: `f4_us1_as1_a_closed_room_answers_room_info_and_the_first_join_creates_it` (`RoomQuery` before `Join` gives `RoomInfo { open: false, settings: defaults, humans: 0 }` and does not take a slot; the first `Join` with `room` makes that player the owner with those settings; the default name is "<owner>'s room"); `f4_us1_as2_a_second_player_joins_the_open_room_and_cannot_change_it` (`RoomInfo` open, `Join.room` ignored, `SetRoom` gives `Error{NotOwner}`, `Welcome` and `Lobby` carry `settings`); `f4_us1_as3_the_owner_changes_the_settings_and_everybody_sees_them` (every client gets a fresh `Lobby` at once, the next match uses them); `f4_us1_as4_settings_are_refused_during_a_match_and_kept_for_the_rematch` (`Error{WrongPhase}`, unchanged settings, same settings after the return to the lobby); `f4_us1_as8_the_room_closes_when_the_last_person_leaves` (then `RoomInfo.open` is false and a new `Join` gets the defaults); `SetRoom` whose `settings` is not an object gives `Error{BadRoom}`; the message sequence of an old client (`Join`, `Welcome`, `Lobby`) is unchanged
- [X] T011 [P] [US1] WebSocket tests in `crates/web/tests/room.rs` and fuzz in `crates/web/tests/fuzz.rs`: two tabs, the first creates a room with `Join.room`, the second sees the same `Lobby.settings`; hostile settings (numbers, arrays, nested objects, a 60 KB name within the 64 KiB limit, control characters) never produce `Malformed` and never affect the other tab; `RoomQuery`, `SetRoom` and `Join.room` are added to the random-frame fuzz
- [X] T012 [P] [US1] Option tests in `crates/web/tests/args.rs` and `crates/cli/tests/args.rs`: `--room-name <text>`, `--mode normal|team|chaser`, `--size small|medium|large`, `--time 2|3|5|10`, `--friendly-fire on|off`, `--no-setup` parse for `eksplode-re-cli host` and (all but `--room-name` and `--no-setup`) for the local game and for `eksplode-re-server`; an invalid value gives a usage error that lists the allowed values (exit code 2); the help text and `--help` mention every option
- [X] T013 [P] [US1] Terminal tests in `crates/cli/tests/setup.rs` and `crates/cli/tests/lobby.rs`: the setup screen (Up and Down select, Left and Right change, typed characters edit the name up to 24, Enter on "Create room" confirms, Esc goes back; friendly fire shown only for Team) and its view text of `contracts/cli-and-page.md`; the lobby first line shows `name - Mode - size - time` and the owner changes the settings with `M`, `S`, `T`, `F` (non-owners cannot); `f4_us1_as7_the_terminal_host_sets_and_shows_the_same_settings` and a terminal player who joins an empty room gets the defaults; `crates/cli/tests/online.rs` keeps working when `Welcome` and `Lobby` carry `settings`
- [X] T014 [P] [US1] Page tests in `crates/web/js-tests/room.test.mjs`: the pure model of `crates/web/assets/room.js` (option lists, defaults, reducer for name, mode, size, time and friendly fire, name limit, payloads for `Join.room` and `SetRoom`, storage under `eksplode.re.room.v1` with corrupt JSON and missing `localStorage` giving the defaults); the reducer in `app.js` shows the creation menu when `RoomInfo.open` is false, the join screen when true, falls back to the join screen after 1 second without an answer, keeps `settings` from `Welcome` and `Lobby`, and shows the `NotOwner`, `WrongPhase` and `BadRoom` messages as notices
- [X] T015 [P] [US1] Consistency test in `crates/web/tests/consistency.rs`: the option lists of `crates/web/assets/room.js` equal the lists of `core::room`, the wire names of `net` and the lists in `docs/PROTOCOL.md`

### Implementation for User Story 1

- [X] T016 [US1] Implement `crates/net/src/room.rs` (new, exported from `crates/net/src/lib.rs`): `RoomSettings::from_wire` and the wire form of `Mode` and `BoardSize` as T009; extend `crates/net/src/protocol.rs`: `ClientMsg::RoomQuery`, `ClientMsg::SetRoom { settings }`, `ClientMsg::Join` gets `room: Option<serde_json::Value>` (`#[serde(default)]`), `ServerMsg::RoomInfo { open, settings, humans }`, `Welcome` and `Lobby` get `settings: RoomSettings` (`#[serde(default)]`), error code `BadRoom`; keep `Client::connect` in `crates/net/src/client.rs` sending no room
- [X] T017 [US1] Implement the room in `crates/net/src/host.rs`: `HostConfig` gets `defaults: RoomSettings`; `RoomQuery` is allowed before `Join` and answered without a slot; the first join into an empty room creates it with `Join.room` or the defaults (empty name becomes "<owner>'s room"); `SetRoom` in the lobby by the owner only (`NotOwner`, `WrongPhase`, `BadRoom`), always sanitized, broadcast as `Lobby`; the match `Config` is `settings.to_config(&base)` at each start; the settings are kept for the rematch and forgotten when the last human leaves the lobby; `Lobby` in `crates/core/src/lobby.rs` gets `settings` and `can_start(&settings)` (Normal: at least 1 participant as today; Team and Chaser: exactly 4 participants) with the missing count in the `CannotStart` message
- [X] T018 [P] [US1] Implement the options in `crates/web/src/options.rs` and `crates/web/src/args.rs` (server start-up defaults of the menu) and in `crates/cli/src/args.rs` (all of T012), validated like the other options, and pass them to `HostConfig.defaults` in `crates/cli/src/app.rs`
- [X] T019 [P] [US1] Implement `crates/web/assets/room.js`: option lists, defaults, reducer, payload builders for `Join.room` and `SetRoom`, `loadRoom()` and `saveRoom()` with `eksplode.re.room.v1` and safe fallbacks
- [X] T020 [US1] Page integration in `crates/web/assets/index.html`, `crates/web/assets/style.css` and `crates/web/assets/app.js`: send `RoomQuery` when the socket opens; the creation panel `#room-panel` (name field, mode cards with a one-line rule each, size and time selectors, friendly-fire switch shown for Team, "Create room" button, keyboard accessible, usable on a phone-sized window) shown before the join screen when the room is closed; the lobby shows a settings summary to everybody and the same controls to the owner (every change sends `SetRoom`, disabled during a match); import `/room.js` (add the route to `crates/web/src/assets.rs` and the route tests in `crates/web/tests/http.rs`; the page stays under 500 KB)
- [X] T021 [US1] Terminal integration in `crates/cli/src/lobby.rs`, `crates/cli/src/view.rs`, `crates/cli/src/online.rs` and `crates/cli/src/app.rs`: the setup screen of `contracts/cli-and-page.md` before the lobby of `host` unless a settings option or `--no-setup` is given, the lobby settings line and owner keys `M`, `S`, `T`, `F`, the settings kept in `Online` from `Welcome` and `Lobby`; make `scripts/pty-enter-check.py` pass `--no-setup`
- [X] T022 [P] [US1] Update `docs/PROTOCOL.md` (`RoomQuery`, `RoomInfo`, `SetRoom`, `Join.room`, `settings` in `Welcome` and `Lobby`, `BadRoom`, the settings wire table), `docs/WEB.md` (creation menu, lobby controls, `/room.js`, storage key) and the README options so the docs tests (`crates/web/tests/docs.rs`) list every new item
- [X] T023 [US1] Extend `scripts/browser-smoke.mjs` to create a room through the menu, join with a second page, change a setting as the owner and save screenshots of the menu and the lobby; run T009-T015, the gates and the smoke script; tick the US1 manual items in `specs/004-game-modes/checklists/manual-004.md`. Commits: `core: add room settings and mode types`, `net: add room query, settings and SetRoom`, `web: add the room creation menu`, `cli: add room options and setup screen`

**Checkpoint**: the room menu works on the page and in the terminal with Normal rules; US1 AS1-AS4, AS7 and AS8 pass

---

## Phase 4: User Story 2 - The end pattern in Normal mode (Priority: P2)

**Goal**: in Normal mode blocks announced for one second fall in a clockwise spiral from the outer ring during the last 60 seconds until at most one player is left; the round no longer ends in a time draw.

**Independent Test**: start a Normal match with 2 minutes and one bot that stands still (dev panel), wait for the last 60 seconds and watch the marked squares, the landings, the crush and the end of the round; play the same seed twice and compare.

### Tests for User Story 2 (write first, must fail)

- [X] T024 [P] [US2] Pattern order and timing tests in `crates/core/tests/closing.rs`: the order is the clockwise spiral of `contracts/modes-and-rules.md` over the non-wall inner squares (hand-checked on the 15 x 13 arena with no blocks); `T0 = round_limit_ticks - closing_ticks`; `interval = max(1, closing_ticks / N)`; each announced square carries `ticks_left = closing_lead_ticks` and counts down; `PatternStarted` once and `BlockWarned` per square; squares that are already walls are skipped without using an interval; every inner square is covered within `closing_ticks + closing_lead_ticks` after `T0` on 15 x 13, 19 x 15 and 25 x 19 (`f4_us2_as1_the_pattern_starts_60_seconds_before_the_end_from_the_outer_ring`); no random numbers are drawn (the state RNG is unchanged by the pattern)
- [X] T025 [P] [US2] Landing tests in `crates/core/tests/closing.rs`: `f4_us2_as2_a_landing_block_crushes_a_player_and_destroys_bombs_and_powerups` (the player gets `PlayerEliminated` in the tick of `BlockLanded`, a bomb on the square disappears without a `BombExploded` and the owner's active bomb count is released, a power-up gets `PowerUpDestroyed`, a flame on it is removed, the tile becomes a `Wall` that a later explosion does not destroy); a flying bomb is unaffected until it lands and a bomb that stops on an announced square is destroyed on landing; an announced square can still be walked on; a bomb that explodes in the tick a block lands still explodes (step order); a power-up under a landing block is destroyed, not collected
- [X] T026 [P] [US2] Round end tests in `crates/core/tests/round_end.rs`: `f4_us2_as3_the_round_goes_on_past_the_time_while_two_players_live`; `f4_us2_as4_the_covered_board_ends_the_round_with_the_last_survivor_or_a_draw` (two players eliminated by the same block give `Draw`); `f4_us2_as5_one_player_left_before_the_pattern_ends_the_round_at_once_and_the_pattern_never_starts`; adapt the existing time-draw tests (`the_round_always_ends_within_the_limit` and similar) to the new rule for Normal and keep the limit draw for a practice round of one participant; `f4_us2_as7_the_same_seed_settings_and_inputs_give_the_same_pattern_and_result` in `crates/core/tests/determinism.rs`
- [X] T027 [P] [US2] Bot tests in `crates/bots/tests/danger.rs` and `crates/bots/tests/behaviour.rs`: the danger map marks an announced square lethal from `ticks_left` onward; a bot standing on or heading into one moves to the nearest safe square in time (scripted scenario with `ticks_left` 20 and a 5-square escape); a path never ends on an announced or landed square; a bot never stays inside a closed pocket during the pattern longer than needed; a soak in `crates/bots/tests/soak.rs` of 50 Normal rounds with 4 bots ends every round and the average survivor count at the end of the pattern is at most 1
- [X] T028 [P] [US2] Drawing tests in `crates/web-render/tests/commands.rs` and `crates/cli/tests/view.rs`: an announced square draws a pulsing red frame and a growing block shadow as `ticks_left` falls (page) and `!!` (terminal); a landed block is an ordinary wall; the status shows the time left and "closing" once the pattern started; `f4_us2_as6_the_announcement_the_landed_blocks_and_the_time_left_are_drawn`
- [X] T029 [P] [US2] Page sound and banner tests in `crates/web/js-tests/sounds.test.mjs`: `warning` on `PatternStarted` and on each `BlockWarned` (rate limited to one per 250 ms), `impact` on `BlockLanded` (louder when a player was crushed in that tick), the banner "The walls close in!" for 2 seconds, all respecting mute and volume; the list of sounds equals the list in `docs/AUDIO.md` (docs test)

### Implementation for User Story 2

- [X] T030 [US2] Implement `crates/core/src/closing.rs` (new, exported from `crates/core/src/lib.rs`): `closing_order(&Arena) -> Vec<Pos>` (clockwise spiral), the start at `T0`, the announce schedule, the countdown and the landing as one function called from `step` after detonation and before power-up collection (`contracts/modes-and-rules.md` step order), only for `Mode::Normal` and `Mode::Team`; wire it into `crates/core/src/step.rs`, extend `eliminate` for landed blocks and change `outcome` so that in these modes the time limit never ends a round with two or more alive (a practice round with one participant keeps the time draw)
- [X] T031 [US2] Implement the bot changes in `crates/bots/src/danger.rs`, `crates/bots/src/pathing.rs` and `crates/bots/src/default_bot.rs` (T027)
- [X] T032 [US2] Draw the pattern in `crates/web-render/src/commands.rs` and `crates/web-render/src/sprites.rs` (announced square, descending block, landing dust, reduced motion respected through the existing setting) and in `crates/cli/src/view.rs`; bump the command buffer layout version in `docs/WEB.md` and its test; rebuild with `./scripts/build-wasm.sh`; the wasm stays at most 300 KB (`sc005_the_whole_page_stays_small`)
- [X] T033 [US2] Implement the sounds and the banner in `crates/web/assets/audio.js`, `crates/web/assets/app.js`, `crates/web/assets/index.html` and `crates/web/assets/style.css` (generated sounds `warning` and `impact`, no recordings) and document them in `docs/AUDIO.md`
- [X] T034 [US2] Create `docs/GAMEPLAY.md` with the Normal mode and the end pattern (order, announcement, landing effects, how the round ends) and link it from the README; extend the docs tests; extend `scripts/browser-smoke.mjs` to run a 2-minute round with the dev panel pausing bots and save screenshots of the announcement and the landed blocks; run T024-T029, the gates and the smoke script; tick the US2 manual items. Commits: `core: add the end pattern`, `bots: avoid announced squares`, `render: draw the end pattern`, `web: add warning and impact sounds`

**Checkpoint**: Normal rounds always end; US2 AS1-AS7 pass

---

## Phase 5: User Story 1, part B - Board size and round time (Priority: P1)

**Goal**: the owner's size and time settings change the match: small 15 x 13, medium 19 x 15, large 25 x 19; 2, 3, 5 or 10 minutes; the page and the terminal fit every size.

**Independent Test**: pick a large board and 10 minutes, start a match, see the large fair arena, the clock starting at 10:00 and the pattern starting at 1:00; check the terminal message on a small terminal.

### Tests for Phase 5 (write first, must fail)

- [X] T035 [P] [US1] Arena fairness tests in `crates/core/tests/arena.rs` for every size and the seeds 1..50: odd dimensions, border and pillar pattern, the four corner squares and their two neighbours free, every non-wall square reachable from every start, blocks never enclose a start; `f4_us1_as6_the_chosen_size_gives_a_fair_reachable_arena_with_the_players_in_the_corners`
- [X] T036 [P] [US1] Settings-to-match tests in `crates/net/tests/room_host.rs`: a started match has the size, the `round_limit_ticks` and the pattern start (`T0`) of the settings for each of the 3 sizes and the 4 times, and a change by the owner before the start is used; the observation `config` carries them (`f4_us1_as6_the_match_uses_the_room_size_and_time`)
- [X] T037 [P] [US1] Fit tests in `crates/web/js-tests/render.test.mjs`, `crates/web-render/tests/view.rs` and `crates/cli/tests/view.rs`: the page view fits 25 x 19 on a 360 px wide canvas and a desktop canvas with tiles of at least 16 px on desktop; the terminal view reports "terminal too small" with the size needed for the chosen board (large needs 52 x 26 characters) and still lets the room settings be changed
- [X] T038 [P] [US1] Latency test in `crates/web/tests/lifecycle.rs`: on the large board with 4 participants, key press to snapshot p95 is at most 200 ms over 200 samples; a 5-minute soak in `crates/bots/tests/soak.rs` on the large board keeps the 20 Hz tick (no tick lasts more than 25 ms on average)

### Implementation for Phase 5

- [X] T039 [US1] Make the sizes and times effective: check `Arena::generate` in `crates/core/src/arena.rs` for fixed assumptions about 15 x 13 (start areas, pillar rule, block density) and fix them so T035 passes; `to_config` already sets size and limit (T006), so only fix what the tests show; enable the size and time selectors in the page (`crates/web/assets/room.js`, `crates/web/assets/app.js`) and keys `S` and `T` in the terminal (`crates/cli/src/lobby.rs`) if they were shown as fixed in Phase 3
- [X] T040 [US1] Fit the page and the terminal: scale the canvas and the camera for the three sizes in `crates/web-render/src/view.rs` and `crates/web/assets/app.js`, the terminal size check in `crates/cli/src/view.rs`; update `docs/GAMEPLAY.md` and `docs/WEB.md` (sizes, times, terminal sizes); extend the smoke script to screenshot all three sizes; run T035-T038, the gates and the smoke script; tick the US1 AS6 manual items. Commits: `core: make the arena fair on all sizes`, `web: fit the three board sizes`, `cli: fit the three board sizes`

**Checkpoint**: sizes and times work; US1 AS1-AS4, AS6-AS8 pass

---

## Phase 6: User Story 3 - Team mode, 2 against 2 (Priority: P3)

**Goal**: 2 against 2 with the Normal rules and the end pattern; team results; friendly fire as a room setting.

**Independent Test**: start a Team match with four bots, eliminate both members of one team and check the winner; check friendly fire on and off with a hand-built state.

### Tests for User Story 3 (write first, must fail)

- [X] T041 [P] [US3] Core tests in `crates/core/tests/team.rs`: `f4_us3_as1_teams_are_fixed_by_slot_and_four_participants_are_needed` (slots 1 and 2 are team 0, slots 3 and 4 are team 1; `can_start` needs exactly 4); `f4_us3_as2_a_team_wins_when_the_other_team_is_eliminated`; `f4_us3_as3_both_teams_eliminated_together_is_a_draw`; `f4_us3_as4_friendly_fire_off_spares_teammates_but_not_the_owner_or_opponents` (including a chain reaction through a teammate's bomb, a flame of a teammate and an enemy on one square, and friendly fire on eliminating a teammate); the end pattern runs in Team mode and the round ends in a `TeamWin`
- [X] T042 [P] [US3] Host tests in `crates/net/tests/room_host.rs`: Team and Chaser rooms are accepted by `SetRoom` once playable; `f4_us1_as5_a_start_with_too_few_participants_is_refused_with_the_missing_count` for Team (3 participants gives `Error{CannotStart}` that names the missing count and says bots can fill the slots) and starts after `AddBot`; teams and roles reach the observation
- [X] T043 [P] [US3] Bot tests in `crates/bots/tests/behaviour.rs`: `f4_us3_as6_a_bot_that_takes_a_slot_plays_for_its_team_and_never_targets_a_teammate` (enemy list excludes teammates; no bomb placed where only a teammate would be hit when friendly fire is on unless cornered; escapes the flames of teammates); a soak of 50 Team rounds with 4 bots ends every round
- [X] T044 [P] [US3] Drawing and text tests in `crates/web-render/tests/commands.rs`, `crates/cli/tests/view.rs`, `crates/web/js-tests/lobby.test.mjs` and `crates/web/js-tests/app.test.mjs`: team ring (team 0 blue, team 1 orange) under the sprite and the letter on the card, team letters `A` and `B` in the terminal status and lobby, the result message names the winning team and its members, a draw says "Draw"; `f4_us3_as5_teams_are_marked_and_the_result_names_the_winning_team`

### Implementation for User Story 3

- [X] T045 [US3] Implement `crates/core/src/team.rs` (new): team of a slot, friendly-fire test using `Flame.owner`, team result; wire into the build of the round (`team` set in Team mode), `eliminate` and `outcome` in `crates/core/src/step.rs` and `crates/core/src/state.rs`; add `Mode::Team` to `PLAYABLE` in `crates/core/src/room.rs`
- [X] T046 [US3] Implement the bot changes in `crates/bots/src/default_bot.rs` and `crates/bots/src/danger.rs` (T043)
- [X] T047 [US3] Draw and show teams: `crates/web-render/src/commands.rs` (rings), `crates/web/assets/app.js` and `crates/web/assets/style.css` (cards, lobby team labels, friendly-fire switch visible for Team, result message), `crates/cli/src/view.rs` and `crates/cli/src/lobby.rs` (letters, lobby teams, result); rebuild with `./scripts/build-wasm.sh`
- [X] T048 [US3] Document Team mode in `docs/GAMEPLAY.md`, `docs/WEB.md` and the README, extend the smoke script with a Team room (screenshots of the lobby and the match), run T041-T044, the gates and the smoke script; tick the US3 manual items. Commits: `core: add Team mode and friendly fire`, `bots: play for a team`, `web: show teams`, `cli: show teams`

**Checkpoint**: Team mode is playable in the page and the terminal; US3 AS1-AS6 and US1 AS5 pass

---

## Phase 7: User Story 4 - Chaser mode (Priority: P4)

**Goal**: one random chaser with 2 bombs and 3 hearts in the center against three runners without bombs on a nearly open arena with random bonuses.

**Independent Test**: start a Chaser match with four bots and check the start, the bonuses, the heart protection and each of the three endings.

### Tests for User Story 4 (write first, must fail)

- [X] T049 [P] [US4] Role and start tests in `crates/core/tests/chaser.rs`: `f4_us4_as1_one_chaser_in_the_center_with_two_bombs_and_three_hearts_and_runners_in_the_corners` (the chaser is drawn from the state RNG after the arena, so the arena of a seed is the same whatever the draw; a different seed can give another chaser; `RoleAssigned` events); the arena keeps pillars only where `(x/2 + y/2) % chaser_pillar_mod == 0`, has a block share near `chaser_block_density`, the center cross and the corners are clear, and every square is reachable on all three sizes; `f4_us4_as2_runners_cannot_place_bombs_and_extra_bomb_and_flame_never_appear`
- [X] T050 [P] [US4] Bonus schedule tests in `crates/core/tests/chaser.rs`: `f4_us4_as3_bonuses_appear_at_a_regular_pace_on_free_squares_up_to_the_cap` (a spawn every `bonus_every_ticks` while fewer than `bonus_cap` lie on the board, on a square with no wall, block, bomb, player or power-up, kinds only from speed 3, kick 2, punch 2, slow 1, hyper speed 1, small bomb 1; over 10000 ticks the pace is within 20% and the weights within 10%); the developer drop table has no effect in this mode; `f4_us4_as7_runners_use_kick_punch_and_speed_as_in_the_other_modes`
- [X] T051 [P] [US4] Hearts and result tests in `crates/core/tests/chaser.rs`: `f4_us4_as4_the_chaser_loses_a_heart_per_explosion_with_two_seconds_of_protection` (own explosion counts, flames during `shield_ticks` are ignored, `HeartLost`, the last heart eliminates and the runners win); `f4_us4_as5_a_runner_dies_from_one_explosion_and_the_chaser_wins_when_the_last_runner_dies`; `f4_us4_as6_the_runners_win_when_the_time_runs_out_and_no_pattern_runs` (including a chaser alone with runners alive at the limit); a punched-back bomb costs the chaser a heart
- [X] T052 [P] [US4] Bot tests in `crates/bots/tests/behaviour.rs` and soak in `crates/bots/tests/soak.rs`: `f4_us4_as8_bots_play_both_roles` (chaser bot walks toward the nearest runner and plants only with a way out and uses the second bomb to cut routes; runner bot flees when the chaser is within 5 squares, picks up bonuses within 6 steps and uses punch and kick on bombs); `soak_chaser`: 200 Chaser rounds with 4 bots end before the limit plus one second, no participant is stuck for more than 5 seconds and bonuses appear at the configured pace within 20% (SC-008)
- [X] T053 [P] [US4] Drawing, HUD and text tests in `crates/web-render/tests/commands.rs`, `crates/cli/tests/view.rs`, `crates/web/js-tests/app.test.mjs`, `crates/web/js-tests/sounds.test.mjs` and `crates/web/js-tests/lobby.test.mjs`: the crown badge above the chaser, the hearts row (blinking while `shield_left > 0`), "Chaser" and "Runner" labels on the cards, hearts as `<3 x2` and the chaser marked `*` in the terminal, the lobby says roles are drawn at the start, the result says "Chaser wins" or "Runners win", the `heart` sound on `HeartLost` under mute and volume; `f4_us4_as9_the_chaser_hearts_and_result_are_shown`
- [X] T054 [P] [US4] Dev panel tests in `crates/net/tests/dev.rs` and `crates/web/js-tests/dev.test.mjs`: `DevSetDrops` is accepted but has no effect in Chaser mode and the panel says so; pausing bots still works

### Implementation for User Story 4

- [X] T055 [US4] Implement `crates/core/src/chaser.rs` (new) and `crates/core/src/arena.rs` (Chaser layout and the clear center): role choice, start state, hearts and shield in `eliminate`, bonus timer and weighted spawn with the state RNG, results `ChaserWins` and `RunnersWin`; wire into `crates/core/src/step.rs` after detonation (bonus step before collection) and the build of the round in `crates/core/src/state.rs`; runners start with `bombs_max = 0`; add `Mode::Chaser` to `PLAYABLE`
- [X] T056 [US4] Implement the Chaser bot roles in `crates/bots/src/default_bot.rs`, `crates/bots/src/danger.rs` and `crates/bots/src/pathing.rs` (T052)
- [X] T057 [US4] Draw and show roles and hearts: `crates/web-render/src/commands.rs` and `crates/web-render/src/sprites.rs` (badge, hearts, blink), `crates/web/assets/app.js`, `crates/web/assets/audio.js` (generated `heart` sound), `crates/web/assets/index.html`, `crates/web/assets/style.css`, `crates/cli/src/view.rs`; the dev panel note in `crates/web/assets/app.js`; rebuild with `./scripts/build-wasm.sh`; document the sound in `docs/AUDIO.md`
- [X] T058 [US4] Document Chaser mode in `docs/GAMEPLAY.md`, `docs/WEB.md` and the README, extend the smoke script with a Chaser room (screenshots of the start, the hearts and the result), run T049-T054, the gates and the smoke script; tick the US4 manual items. Commits: `core: add Chaser mode`, `bots: play the chaser and runner roles`, `render: draw the chaser and hearts`, `web: show roles and hearts`, `cli: show roles and hearts`

**Checkpoint**: all three modes work in both interfaces; US4 AS1-AS9 pass

---

## Phase 8: Polish and cross-cutting concerns

- [X] T059 [P] Fuzz and hostile input in `crates/net/tests/codec.rs` and `crates/web/tests/fuzz.rs`: random and truncated `RoomQuery`, `SetRoom`, `Join.room` and `RoomInfo` frames, 64 KiB and 64 KiB + 1 messages, never a panic and never a changed room (FR-029)
- [X] T060 [P] Compatibility tests in `crates/net/tests/loopback.rs` and `crates/cli/tests/online.rs`: an old-style client that never sends `RoomQuery` or `SetRoom` joins, plays a Normal match and sees an unchanged message sequence; a client that does not know `RoomInfo`, `PatternStarted` or `HeartLost` still decodes the rest (FR-028)
- [X] T061 [P] Determinism across modes in `crates/core/tests/determinism.rs`: for each mode and size the same seed, settings and inputs give identical event lists and outcomes over a full round
- [X] T062 [P] Complete the docs: `docs/GAMEPLAY.md` (all modes, controls, options), `docs/PROTOCOL.md`, `docs/WEB.md`, `docs/AUDIO.md`, README (room creation, every setting, every terminal option), extend `crates/web/tests/docs.rs` so each option, message, mode and sound of `contracts/` is checked (FR-030, SC-011)
- [X] T063 Run the whole gate and the quickstart of `specs/004-game-modes/quickstart.md`: `cargo fmt --all -- --check`, `cargo clippy --workspace --all-targets -- -D warnings`, `cargo test --workspace`, `node --test crates/web/js-tests`, `scripts/check-scenarios.sh`, `python3 scripts/pty-enter-check.py --attempts 20`, the browser smoke script; walk the manual acceptance steps and complete `specs/004-game-modes/checklists/manual-004.md`; note the measured wasm and page sizes and the large-board latency in `docs/DEVELOPMENT.md` (SC-010)

---

## Dependencies and execution order

- Phase 1 has no dependency. Phase 2 needs Phase 1 and blocks everything else.
- Phase 3 (US1) needs Phase 2. Phase 4 (US2) needs Phase 2 and is independent of Phase 3 at the rule level (T024-T027, T030, T031); its page parts (T029, T033) need the page of Phase 3 only for the room flow in the smoke script.
- Phase 5 (US1 part B) needs Phases 3 and 4 (the pattern timing is checked on every size).
- Phase 6 (US3) needs Phases 2, 3 and 4. Phase 7 (US4) needs Phases 2, 3 and 5 and does not need Phase 6.
- Phase 8 needs the phases whose features it tests; T063 comes last.

Story order: US1 (Phases 3, 5) then US2 (Phase 4) then US3 (Phase 6) then US4 (Phase 7); the delivery order is Phases 3, 4, 5, 6, 7 (menu, end pattern, other settings, new modes).

## Parallel opportunities

- Phase 1: T002 and T003 together after T001.
- Phase 2: T005 and T008 in parallel (T004 first, T006 and T007 in order).
- Phase 3: the test tasks T009-T015 are all independent; after T016 and T017, T018, T019 and T022 run in parallel, then T020 and T021.
- Phase 4: T024-T029 are independent; T031, T032 and T033 run in parallel once T030 is merged.
- Phase 5: T035-T038 are independent.
- Phase 6: T041-T044 are independent; T046 and T047 run in parallel after T045.
- Phase 7: T049-T054 are independent; T056 and T057 run in parallel after T055.
- Phase 8: T059-T062 are independent.

## Implementation strategy

- MVP: Phases 1 to 3 (the room creation menu with Normal rules); the game plays exactly as before with a described room (SC-003).
- Then ship in the requested order: end pattern (Phase 4), sizes and times (Phase 5), Team (Phase 6), Chaser (Phase 7), polish (Phase 8).
- After every phase run the common gate of the quickstart and commit by part; never leave a mode selectable before its rules exist (`PLAYABLE`).
- Tuning values (`closing_lead_ticks`, `shield_ticks`, `bonus_every_ticks`, `bonus_cap`, densities) live in `Config` so they can change without changing a rule.
