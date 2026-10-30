---

description: "Task list for feature 012: bot behaviours and difficulty levels"
---

# Tasks: Bot Behaviours and Difficulty Levels

**Input**: Design documents from `specs/012-bot-behaviours-difficulty/`

**Prerequisites**: [plan.md](plan.md), [spec.md](spec.md), [research.md](research.md),
[data-model.md](data-model.md), [contracts/](contracts/), [quickstart.md](quickstart.md)

**Tests**: Required. The constitution (Principle II) mandates test-first: every test task below MUST
be written and observed failing before its implementation task. Test names follow
`f12_us<N>_as<M>_<description>` for acceptance scenarios.

**Organization**: Tasks are grouped by user story. All paths are relative to `game/`.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies on incomplete tasks)
- **[Story]**: US1 (behaviours + developer override, P1), US2 (difficulty, P2), US3 (host-only room setting & UI, P3)

---

## Phase 1: Setup

**Purpose**: Confirm a clean baseline before changing bot behaviour.

- [X] T001 Run `cargo fmt --check`, `cargo clippy --all-targets -- -D warnings`, `cargo test --workspace` and `node --test crates/web/js-tests/` from `game/` and record any pre-existing failures in `specs/012-bot-behaviours-difficulty/quickstart.md` under a "Baseline" note (no code changes)

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: The `BotDifficulty` value, the `RoomSettings` field and the `Profile` plumbing every story uses.

**⚠️ CRITICAL**: No user story work can begin until this phase is complete.

- [X] T002 [P] Write failing tests in `crates/core/tests/room.rs`: `BotDifficulty::ALL == [Easy, Medium, Hard]`; `name()` is `"easy"|"medium"|"hard"`, `from_name` round-trips and rejects `"HARD"`; `label()` is `Easy|Medium|Hard`; `BotDifficulty::default() == Medium`; `RoomSettings::default().bot_difficulty == Medium`; serde JSON of `RoomSettings` contains `"bot_difficulty":"medium"` and JSON without the field deserializes to Medium; `step_bot_difficulty(1)` from Hard wraps to Easy and `step_bot_difficulty(-1)` from Easy wraps to Hard; `to_config` output is unaffected by `bot_difficulty`. Update the existing field-list assertion near line 165 to include `"bot_difficulty"`
- [X] T003 Implement `BotDifficulty` in `crates/core/src/room.rs` (variants `Easy`, `Medium` (`#[default]`), `Hard`; `#[cfg_attr(feature = "serde", serde(rename_all = "lowercase"))]`; `ALL`, `name`, `from_name`, `label` mirroring `BoardSize`), add `RoomSettings.bot_difficulty: BotDifficulty` with `#[cfg_attr(feature = "serde", serde(default))]` and doc comment "The difficulty of every bot of the room's matches; Medium by default.", set it to `Medium` in `RoomSettings::default()`, add `step_bot_difficulty(step: i8)` using `wrap`, keep `to_config` unchanged, and re-export `BotDifficulty` from `crates/core/src/lib.rs`; make T002 pass and fix any struct-literal compile errors in existing tests by adding `..RoomSettings::default()` or the field
- [X] T004 [P] Write failing tests in `crates/net/tests/room.rs` for `from_wire`: `"bot_difficulty":"hard"` → Hard; missing field → Medium; `"HARD"`, `3`, `null`, `[]` → Medium, never an error (contracts/room-settings.md rule 1)
- [X] T005 Parse `bot_difficulty` in `from_wire` in `crates/net/src/room.rs` with `field("bot_difficulty").and_then(|v| serde_json::from_value::<BotDifficulty>(v).ok()).unwrap_or(defaults.bot_difficulty)`; make T004 pass
- [X] T006 Create `crates/bots/src/profile.rs` with `pub enum Behaviour { Aggressive, Protective, Balanced }` (+ `ALL`), `pub struct Skill { pub think_every: u32, pub react_ticks: u32, pub mistake_chance: f32 }`, `pub struct Profile { pub behaviour: Behaviour, pub difficulty: BotDifficulty }` with `Default` = Balanced / Hard; declare the module and re-export `Behaviour`, `Profile`, `Skill` in `crates/bots/src/lib.rs` (pick/name/of logic comes in US1/US2)
- [X] T007 Add `profile: Profile` to `DefaultBot` in `crates/bots/src/default_bot.rs`, a `pub fn with_profile(seed: u64, profile: Profile) -> Self`, `pub fn profile(&self) -> Profile`, and make `new(seed)` call `with_profile(seed, Profile::default())`; doc comment on `new`: "Balanced behaviour at Hard difficulty, the bot from before behaviours existed."
- [X] T008 [P] Write a test in `crates/bots/tests/parity.rs`: over 20 seeded four-bot matches (reuse the loop shape of `crates/bots/tests/soak.rs`), `DefaultBot::new(s)` and `DefaultBot::with_profile(s, Profile::default())` return identical `PlayerInput`s every tick (FR-015); it must keep passing through every later phase

**Checkpoint**: `cargo test --workspace` green; room settings carry `bot_difficulty`; bots accept a profile that changes nothing yet.

---

## Phase 3: User Story 1 - Bots play with distinct behaviours (Priority: P1) 🎯 MVP

**Goal**: Each bot gets a random, reproducible behaviour per match; Aggressive, Protective and Balanced play differently while keeping today's safety rules. In developer mode the owner can force the behaviour.

**Independent Test**: Seeded runs show all three behaviours are picked; scripted scenarios show an Aggressive bot heads for a reachable opponent while a Protective bot in the same situation goes for the power-up/block and does not end next to the opponent; a developer-mode room with a forced behaviour creates only bots of that behaviour.

### Tests for User Story 1 (write first, observe failing)

- [X] T009 [P] [US1] In `crates/bots/tests/behaviour.rs` add `f12_us1_as1_pick_gives_each_bot_one_of_three_behaviours` (`Behaviour::pick` over seeds 0..300 returns only `Behaviour::ALL` members, each at least once), `f12_us1_as2_pick_is_reproducible` (same seed → same behaviour; `Profile::pick(s, d).difficulty == d`) and a test that `Behaviour::from_name(b.name()) == Some(b)` for every behaviour with names `"aggressive"|"protective"|"balanced"`, and `from_name("random")`/`from_name("Aggressive")` are `None`
- [X] T010 [P] [US1] In `crates/bots/tests/behaviour.rs` add `f12_us1_as3_aggressive_heads_for_a_reachable_opponent`: open arena, opponent ~8 squares away along a free path, a power-up 2 squares away in the other direction; an Aggressive bot's first move reduces its path distance to a square whose blast would reach the opponent, while a Balanced bot (`DefaultBot::new`) goes for the power-up
- [X] T011 [P] [US1] In `crates/bots/tests/behaviour.rs` add `f12_us1_as4_protective_prefers_powerups_and_keeps_off_opponents`: opponent 3 squares away and a power-up 3 squares away; a Protective bot moves toward the power-up; and in a second state where its only goal square is adjacent to an opponent and another safe move exists, its move does not end adjacent to the opponent (FR-006)
- [X] T012 [P] [US1] In `crates/bots/tests/behaviour.rs` add `f12_us1_behaviours_keep_the_safety_layer`: for each `Behaviour` (at Hard), reuse the existing scripted states for escaping a blast, not self-trapping, Team friendly-fire bomb refusal (`team_state_of(true, …)`), closing-pattern inward move and runner fleeing a chaser, and assert the same safe outcome as the existing Balanced tests (FR-008)
- [X] T013 [P] [US1] In `crates/bots/tests/soak.rs` add `f12_sc001_behaviour_distribution` (300 picks: each behaviour 25%–42%) and `f12_sc004_aggressive_hits_more_protective_survives_longer` (100 seeded four-bot matches, Hard, slots given fixed behaviours: count bombs whose blast tiles contained an opponent at placement and ticks survived per behaviour; Aggressive hits ≥ 1.3 × Protective hits, Protective mean survival > Aggressive mean survival); print the numbers with `--nocapture`
- [X] T014 [P] [US1] In the `#[cfg(test)]` module of `crates/net/src/room.rs` add `f12_us1_as5_rematch_repicks_behaviours` and `f12_us1_as6_takeover_bot_gets_a_picked_behaviour`: after `Start`, each `game.bots[slot]` has `profile() == Profile::pick(seed.wrapping_mul(4).wrapping_add(slot), settings.bot_difficulty)`; after `vacate_connection` mid-match the new bot's profile equals `Profile::pick(matches*8+slot, settings.bot_difficulty)`
- [X] T015 [P] [US1] In the same `#[cfg(test)]` module of `crates/net/src/room.rs` add `f12_us1_as7_dev_forced_behaviour_applies_to_new_bots`: a developer-mode room, owner sends `DevSetBotBehaviour { behaviour: Some("aggressive") }` → after `Start` every bot's `profile().behaviour == Aggressive`, a mid-match takeover bot too; then `behaviour: None` → the next `Start` uses `Behaviour::pick` again
- [X] T016 [P] [US1] In `crates/net/tests/dev.rs` add `f12_us1_as7_dev_behaviour_is_broadcast` (owner in a `--dev` room sends `DevSetBotBehaviour` with `"protective"` → every member receives `DevState` with `"behaviour":"protective"`, also accepted during a match) and `f12_us1_as8_dev_behaviour_is_refused` (non-owner → `NotOwner`; room not in developer mode → `Unsupported`; `"random"`, `"AGGRESSIVE"`, `3` → `BadValue`; `DevState.behaviour` unchanged), for both the v2 room path and the legacy host path
- [X] T017 [P] [US1] In `crates/web/js-tests/dev.test.mjs` add tests: `reduce` of `DevState` with `behaviour: "balanced"` stores it and a missing `behaviour` stores `null`; `devPanel(state)` exposes `behaviour: { value, options: ["random","aggressive","protective","balanced"], disabled }` with `value "random"` for `null`, `disabled` for non-owners; the panel action builder returns `{type: "DevSetBotBehaviour", behaviour: "aggressive"}` and `behaviour: null` for `"random"`

### Implementation for User Story 1

- [X] T018 [US1] In `crates/bots/src/profile.rs` implement `Behaviour::pick(seed)` as `Behaviour::ALL[Rng::new(seed ^ BEHAVIOUR_SALT).next_below(3) as usize]` (a private `const BEHAVIOUR_SALT: u64` with a one-line comment that it keeps picks off the decision RNG stream), `Behaviour::name`/`from_name`/`label` (`"aggressive"|"protective"|"balanced"`, `Aggressive|Protective|Balanced`) and `Profile::pick(seed, difficulty)`; T009 passes
- [X] T019 [US1] Refactor the goal section of `DefaultBot::decide` in `crates/bots/src/default_bot.rs` (after `can_bomb_here`) into per-behaviour priority order per research R3: Balanced = power-up(8) → hunt(14) → dig(14) → wander (unchanged, T008 must stay green); Aggressive = hunt(20) → power-up(4) → dig(14) → approach nearest enemy (path to a square adjacent to the nearest `is_enemy` player, depth 20) → wander; Protective = power-up(8) → dig(14) → hunt(5) → keep-away → wander
- [X] T020 [US1] For Protective in `crates/bots/src/default_bot.rs`: reject goal squares adjacent (Manhattan 1) to an enemy in power-up/dig/hunt predicates; add `keep_away` (when the nearest enemy is within 3, step to a passable, non-malus neighbour with `danger.safe_from(t, 1)` that maximises Manhattan distance to it); use `FLEE_DISTANCE + 2` in `flee_chaser` for Protective runners; T010–T012 pass
- [X] T021 [US1] In `crates/net/src/protocol.rs` add v2 `Request::DevSetBotBehaviour { request_id, room_id, behaviour: Option<String> }` (include it in the `request_id`/`room_id` accessor matches next to `DevPauseBots`), legacy `ClientMsg::DevSetBotBehaviour { behaviour: Option<String> }`, and `behaviour: Option<String>` with `#[serde(default)]` on both `Reply::DevState` and `ServerMsg::DevState` (update the `Reply → ServerMsg` conversion near line 203); map the legacy message to the v2 request in `crates/net/src/client.rs` next to `DevPauseBots`; add a protocol round-trip test alongside the existing `Dev*` codec tests in `crates/net/tests/codec.rs`
- [X] T022 [US1] Store `bot_behaviour: Option<Behaviour>` (default `None`) in the room (`crates/net/src/room.rs`, next to `bots_paused`, copied where `bots_paused` is copied near line 880) and the legacy host (`crates/net/src/host.rs`, new `DevCommand::Behaviour(Option<Behaviour>)` handled in `dev_command`); handle `DevSetBotBehaviour` in the existing dev-command branch (owner via `self.owner(..)`, `Unsupported` when `!self.dev`, `Behaviour::from_name` failure → `BadValue`); include `behaviour: self.bot_behaviour.map(|b| b.name().to_string())` in every `DevState` (`dev_state()`, the dev-command reply, the legacy `ServerMsg::DevState` near line 2208); T016 passes
- [X] T023 [US1] Replace bot creation with profiles, using the forced behaviour when set: a helper per host computing `Profile { behaviour: self.bot_behaviour.unwrap_or_else(|| Behaviour::pick(s)), difficulty: self.lobby.settings.bot_difficulty }`; `crates/net/src/room.rs` Start (`s = seed.wrapping_mul(4).wrapping_add(index as u64)`) and `vacate_connection` takeover (`s = self.matches.get().wrapping_mul(8).wrapping_add(slot as u64)`); `crates/net/src/host.rs` `bots_for` (pass the override and difficulty) and the disconnect takeover near line 2482; `crates/cli/src/app.rs` `make_bots` (random pick, lobby settings' difficulty, also on `rematch`); T014, T015 pass
- [X] T024 [US1] Add the "Bot behaviour" `<select id="dev-behaviour">` (options Random/Aggressive/Protective/Balanced, values `random|aggressive|protective|balanced`) to `#dev-panel` in `crates/web/assets/index.html`; in `crates/web/assets/app.js` store `behaviour` from `DevState` in `devState`, extend `devPanel(state)` and the dev action builder (~line 422) per T017, render the select's value/disabled state where the pause button is rendered, and send the action on `change`; keep keyboard/controller focus order in the dev panel working; T017 passes
- [X] T025 [US1] Make T013 pass, tuning only Aggressive/Protective depths in `crates/bots/src/default_bot.rs` if needed (Balanced untouched); then run `cargo test --workspace` and fix `net`/`web`/`cli` integration tests broken by random behaviours: tests that need exact bot moves run their room in developer mode with `DevSetBotBehaviour { behaviour: Some("balanced") }` and `bot_difficulty: "hard"` (today's bot, FR-015); other tests assert outcomes that do not depend on bot moves

**Checkpoint**: US1 complete — bots differ by behaviour in every client, developer mode can force a behaviour; difficulty still has no effect.

---

## Phase 4: User Story 2 - Bots play at a chosen difficulty (Priority: P2)

**Goal**: Easy/Medium bots decide less often, notice opponents' new bombs late and make seeded mistakes; Hard is unchanged.

**Independent Test**: Scripted reaction-delay scenarios plus seeded 1v1 win rates per difficulty pair.

### Tests for User Story 2 (write first, observe failing)

- [X] T026 [P] [US2] In `crates/bots/tests/behaviour.rs` add `f12_skill_table_is_ordered`: `Skill::of(Easy) == {6, 10, 0.20}`, `Skill::of(Medium) == {3, 5, 0.06}`, `Skill::of(Hard) == {1, 0, 0.0}`, and Easy ≥ Medium ≥ Hard on every field (data-model.md "Skill")
- [X] T027 [P] [US2] In `crates/bots/tests/behaviour.rs` add `f12_us2_as1_hard_reacts_at_once`: an opponent's bomb placed this tick (`ticks_left == config.fuse_ticks`) with the bot on its blast line → a Hard bot's input moves out of the line immediately
- [X] T028 [P] [US2] In `crates/bots/tests/behaviour.rs` add `f12_us2_as2_easy_and_medium_react_late`: same state, bomb age `< react_ticks` → an Easy (and a Medium) bot whose fresh-decision tick falls in that window does not flee; with bomb age `>= react_ticks` it flees; the bot's own bombs are seen at age 0; flames are always seen
- [X] T029 [P] [US2] In `crates/bots/tests/behaviour.rs` add `f12_us2_think_interval_holds_the_last_move`: an Easy bot fed consecutive observations returns a fresh decision on the first tick, then repeats only its last `mv` (never `bomb`/`punch`) for the next `think_every - 1` ticks, and drops the held move (no input) when the next square is burning or not passable
- [X] T030 [P] [US2] In `crates/bots/tests/behaviour.rs` add `f12_us2_as4_hard_makes_no_mistakes` (Balanced/Hard via `with_profile` matches `DefaultBot::new` input-for-input across a seeded match — complements T008) and `f12_us2_as5_mistakes_never_step_into_fire`: over 200 seeds, Easy bots in states with burning neighbours never return a move whose target square has a `Flame` (FR-013, no exception)
- [X] T031 [P] [US2] In `crates/bots/tests/behaviour.rs` add `f12_us2_difficulty_is_reproducible`: two Easy bots with the same seed and profile fed the same observations return identical inputs (FR-014)
- [X] T032 [P] [US2] In `crates/bots/tests/soak.rs` add `f12_sc002_win_rates_by_difficulty` (100 seeded 1v1 matches per pair, same behaviour on both sides, sides swapped every other seed: Hard vs Easy ≥ 65% of decided rounds, Medium vs Easy ≥ 55%, Hard vs Medium ≥ 55%) and `f12_sc003_easy_self_eliminates_more` (100 seeded four-bot matches per difficulty: Easy self-eliminations > 0 and ≥ 2 × Hard); print the numbers

### Implementation for User Story 2

- [X] T033 [US2] Implement `Skill::of(BotDifficulty)` in `crates/bots/src/profile.rs` with the values in data-model.md; T026 passes
- [X] T034 [US2] Add `skill: Skill`, `last: PlayerInput` and `since_think: u32` to `DefaultBot` in `crates/bots/src/default_bot.rs`; at the top of `decide` (after the dead/not-Playing early return) hold `last.mv` while `since_think + 1 < skill.think_every`, dropping it if the target square has a flame or is not `passable`; reset `since_think` on a fresh decision; at Hard `think_every == 1` so behaviour is unchanged; T029 passes
- [X] T035 [US2] Implement the reaction delay in `crates/bots/src/default_bot.rs`: when `skill.react_ticks > 0`, decide on a cloned `Observation` whose `bombs` excludes bombs with `owner != obs.me` and `obs.config.fuse_ticks.saturating_sub(b.ticks_left) < skill.react_ticks`; never clone at Hard; T027, T028 pass
- [X] T036 [US2] Implement mistakes in `crates/bots/src/default_bot.rs` per data-model.md flow: only when `skill.mistake_chance > 0.0`, roll `self.rng.chance(skill.mistake_chance)` on a fresh decision; planned bomb → no input; in danger (`!danger.safe_from(me.pos, 1)`) → no input; otherwise 50/50 hesitate or wrong turn (random `shuffled_dirs` neighbour that is `passable` and has no flame); store the result in `last` with `bomb`/`punch` cleared; T030, T031 pass
- [X] T037 [US2] Run `cargo test -p eksplode-re-bots --test soak -- --nocapture`; if T032 fails, tune `Skill` values in `crates/bots/src/profile.rs` (keeping Easy ≥ Medium ≥ Hard and Hard = `{1, 0, 0.0}`), update the table in `specs/012-bot-behaviours-difficulty/research.md` R4 and `data-model.md` to the final values, and adjust T026 accordingly

**Checkpoint**: US2 complete — the room's (default Medium) difficulty now changes how bots play in every client.

---

## Phase 5: User Story 3 - The room owner chooses the bot difficulty (Priority: P3)

**Goal**: Only the host (room owner) changes the difficulty, only in the lobby, in the web page and the terminal host; everyone sees it. Terminal local games and the headless web server have no control (Medium).

**Independent Test**: Change the difficulty as owner, start a match, check bots' profiles; a non-owner or in-match change is refused.

### Tests for User Story 3 (write first, observe failing)

- [X] T038 [P] [US3] In `crates/core/tests/room.rs` add `f12_us3_label_shows_bot_difficulty`: `RoomSettings::default()` with name "Friday" labels as `Friday - Normal - 15 x 13 - 3 min - Medium bots`; Team mode keeps the friendly-fire part before ` - <Label> bots`
- [X] T039 [P] [US3] In `crates/net/tests/room_host.rs` add `f12_us3_as1_new_room_defaults_to_medium` (CreateRoom without the field → settings report `"bot_difficulty":"medium"`), `f12_us3_as2_owner_sets_difficulty_and_members_see_it` (owner `SetRoom` with `"hard"` → every member's `RoomUpdated` shows `"hard"`), `f12_us3_as3_non_owner_cannot_change_it` (`NotOwner`, value unchanged), `f12_us3_as4_no_change_during_a_match` (refused with the existing in-match code), `f12_us3_as6_old_clients_get_medium` (settings JSON without the field from an existing helper in `crates/net/tests/common/mod.rs`); these tests assert only wire-visible results
- [X] T040 [P] [US3] In the `#[cfg(test)]` module of `crates/net/src/room.rs` add `f12_us3_as2_start_uses_room_difficulty` (owner sets Hard, then `Start` → every `game.bots[slot].profile().difficulty == Hard`) and `f12_us3_as5_takeover_bot_uses_room_difficulty` (room at Easy, a human leaves mid-match → the new bot's `profile().difficulty == Easy`)
- [X] T041 [P] [US3] In `crates/cli/tests/args.rs` add tests: `host --bot-difficulty easy` sets `room.bot_difficulty`; default is Medium; `--bot-difficulty extreme` and a missing value are usage errors; `--bot-difficulty` alone skips the host setup screen like `--mode`; `local --bot-difficulty hard` is a usage error (only the host chooses, FR-021) (contracts/room-settings.md)
- [X] T042 [P] [US3] In `crates/cli/tests/setup.rs` add `f12_us3_as7_setup_screen_has_a_bot_difficulty_row`: the row is visible in every mode, Right from Medium gives Hard, Left from Easy wraps to Hard
- [X] T043 [P] [US3] In `crates/cli/tests/online.rs` add `f12_us3_as7_owner_cycles_difficulty_with_d` (owner presses `d` → a `SetRoom` with the next difficulty is sent and the lobby shows it) and `f12_us3_as3_non_owner_d_is_refused` (notice "Only the room owner can change the bot difficulty", nothing sent), following the existing `m`/`s`/`t`/`f` tests near line 907
- [X] T044 [P] [US3] In `crates/web/js-tests/room.test.mjs` update `defaultRoom()` and `sanitizeRoom` expectations to include `bot_difficulty: "medium"`; add: `sanitizeRoom({bot_difficulty: "hard"})` keeps `"hard"`, `"HARD"`/`3`/missing → `"medium"`; `roomReducer(r, {type: "bot_difficulty", value: "easy"})` sets it and ignores invalid values; the summary line ends with `Medium bots`. Update fixtures in `crates/web/js-tests/lobby.test.mjs` that build settings objects
- [X] T045 [P] [US3] In `crates/web/js-tests/lobby.test.mjs` (or `app.test.mjs`, where the lobby configuration panel is tested today) add: the lobby panel's bot-difficulty select reflects the room's value, is enabled for the owner in the lobby, and disabled for non-owners and during a match (existing `roomControls()` gating)

### Implementation for User Story 3

- [X] T046 [US3] Append ` - {bot_difficulty.label()} bots` at the end of `RoomSettings::label()` in `crates/core/src/room.rs` (after the Team friendly-fire part); update the doc comment example; T038 passes and fix existing label assertions across `core`/`cli`/`net`/`web` tests
- [X] T047 [US3] Verify owner/lobby enforcement in `crates/net/src/room.rs` and `crates/net/src/host.rs` `SetRoom` handling needs no change (the whole settings object is already validated); add code only if T039 shows a gap; T039, T040 pass
- [X] T048 [P] [US3] Add `--bot-difficulty <easy|medium|hard>` to `host` parsing only in `crates/cli/src/args.rs` using `BotDifficulty::from_name` (error variant consistent with `--friendly-fire`), count it among the room-setting flags that skip the host setup screen, leave `local` rejecting it as an unknown option, and document it in the host usage text (lines ~14–53), including the `d` lobby key and the updated `--dev` description ("pause bots, choose the bot behaviour and set drop chances"); T041 passes
- [X] T049 [P] [US3] Add a `BOT_DIFFICULTY` row to `crates/cli/src/setup.rs` (always visible, before `CREATE`; Left/Right → `step_bot_difficulty(±1)`; rendered as `Bot difficulty  < Medium >`); T042 passes
- [X] T050 [P] [US3] Handle `d`/`D` in the owner lobby key match in `crates/cli/src/online.rs` (~line 725): owner → `settings.step_bot_difficulty(1)` then `ClientMsg::set_room(&settings)`; non-owner → notice "Only the room owner can change the bot difficulty"; T043 passes
- [X] T051 [US3] In `crates/web/assets/room.js` add `bot_difficulty: "medium"` to `defaultRoom()`, validate it in `sanitizeRoom` against `["easy", "medium", "hard"]`, handle `{type: "bot_difficulty"}` in `roomReducer`, and append `${Label} bots` to the summary parts; T044 passes
- [X] T052 [US3] In `crates/web/assets/index.html` add a labelled "Bot difficulty" `<select>` (options Easy/Medium/Hard, values `easy|medium|hard`) to the create-room form next to the friendly-fire control and to the lobby configuration panel; in `crates/web/assets/app.js` register them in `el` (e.g. `el.roomBots`, `el.lobbyBots`), set their values where `el.roomFf`/`el.lobbyFf` are set (~lines 1608, 1663), wire `change` listeners to `editRoom`/`editRoomConfig` with `{type: "bot_difficulty", value}` (~lines 1686, 1714), and apply the same owner/phase disabling as the other panel controls; keep keyboard/controller focus order working (features 005/006); T045 passes

**Checkpoint**: All three stories work end to end in web and terminal host.

---

## Phase 6: Polish & Cross-Cutting Concerns

- [X] T053 [P] Document behaviours and the difficulty setting in `README.md` ("Bots" section and the room-settings bullets), the host-only `--bot-difficulty` flag, the `d` lobby key and the `--dev` bot-behaviour option
- [X] T054 [P] Update `docs/GAMEPLAY.md`: add "Bot difficulty | Easy, Medium or Hard | Medium" to the room settings table (host/owner only), mention the `d` terminal key, and describe the three behaviours (random per match) and what each difficulty changes
- [X] T055 [P] Update `docs/PROTOCOL.md`: room settings example and table with `bot_difficulty` (values, default, fallback rule) per contracts/room-settings.md; `DevSetBotBehaviour` in the client message table and `behaviour` in the `DevState` row; and `docs/DEVELOPMENT.md` developer-mode section with the behaviour override; run `cargo test -p eksplode-re-web --test docs`
- [X] T056 Run `cargo fmt --check`, `cargo clippy --all-targets -- -D warnings`, `cargo test --workspace`, `node --test crates/web/js-tests/` and `cargo test -p eksplode-re-bots --test soak -- --nocapture` from `game/`; fix any failure
- [X] T057 Walk through the manual checks in `specs/012-bot-behaviours-difficulty/quickstart.md` (terminal host, host setup, web owner/non-owner, compatibility, developer mode) and run `node scripts/browser-smoke.mjs` against a local server to confirm the create form, lobby panel and dev panel render with the new selects
- [X] T058 Confirm every acceptance scenario in `specs/012-bot-behaviours-difficulty/spec.md` maps to a named `f12_us*_as*` test (US1 AS1–AS8, US2 AS1–AS5, US3 AS1–AS7) and update spec/plan if the final `Skill` values or priorities deviated

---

## Implementation Notes

- T013/SC-004 compares 100 all-Aggressive matches with 100 all-Protective matches on the same seeds
  (mixed matches measure who wins, not how a behaviour plays); Protective steps back from close
  threats before bombing (research R3).
- T016: the network tests go through the room protocol (the SDK client always negotiates it); the
  legacy host handles `DevSetBotBehaviour` the same way but has no separate network test.
- T043: a non-owner's `d` gets the server's "Only <owner> …" refusal, like `m`/`s`/`t`/`f`.
- T048: the terminal setup screen gained a row, so creating a room from the terminal directory takes
  six Enters (`crates/cli/tests/online.rs`, `scripts/pty-enter-check.py` updated).
- T057: `scripts/browser-smoke.mjs` already failed before this feature (it waits for the `#join`
  button of the pre-011 page); the new controls were checked in a browser instead (create form,
  lobby panel, dev panel, a running match with no console errors).
- The docs test (`crates/web/tests/docs.rs`) now also checks the bot difficulty flag, names and the
  three behaviours in the guides.

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: none
- **Foundational (Phase 2)**: after Setup; blocks all stories (T003 before T005/T006; T006 before T007; T007 before T008)
- **US1 (Phase 3)**: after Foundational; T018 → T019 → T020 (same file); T021 → T022 → T023 (protocol, then dev state, then bot creation using it); T024 after T021; T025 last
- **US2 (Phase 4)**: after Foundational; touches `default_bot.rs` like US1, so in practice after T019–T020 to avoid conflicts; its tests (T026–T032) can be written in parallel with US1 implementation
- **US3 (Phase 5)**: after Foundational; T040's Start assertion needs T023
- **Polish (Phase 6)**: after the stories that are shipped

### User Story Dependencies

```mermaid
flowchart LR
  F[Phase 2 Foundational] --> US1[US1 Behaviours + dev override P1]
  F --> US2[US2 Difficulty P2]
  F --> US3[US3 Host-only room setting P3]
  US1 -. T023 bot creation .-> US3
  US1 -. same file default_bot.rs .-> US2
```

### Within Each User Story

- Tests first, observed failing → implementation → story checkpoint
- `crates/bots/src/profile.rs` before `crates/bots/src/default_bot.rs` changes
- Protocol before host state before bot wiring before UI

## Parallel Opportunities

- Phase 2: T002, T004 in parallel; T008 after T007
- US1: T009–T017 all in parallel (different tests/files, except T014/T015 share the room.rs test module — write them together); T018 → T019 → T020 sequential (same file) in parallel with T021 → T022 → T023; T024 in parallel with T022/T023
- US2: T026–T032 in parallel; T033 → T034 → T035 → T036 sequential (same file)
- US3: T038–T045 in parallel; T048, T049, T050 in parallel (different files); T051 → T052
- Polish: T053–T055 in parallel

### Parallel Example: User Story 1

```text
T009–T013 crates/bots/tests/{behaviour.rs,soak.rs}
T014+T015 crates/net/src/room.rs (test module)
T016 crates/net/tests/dev.rs | T017 crates/web/js-tests/dev.test.mjs
then
T018 → T019 → T020 (bots)  ||  T021 → T022 → T023 (net)  ||  T024 (web dev panel, after T021)
```

## Implementation Strategy

### MVP First (User Story 1)

1. Phase 1 + Phase 2
2. Phase 3 (US1): random behaviours in every client plus the developer override, difficulty inert
3. Stop and validate: T013 statistics, play a hosted match (`cargo run -p eksplode-re-cli -- host --bots 3`), force a behaviour in a `--dev` web room

### Incremental Delivery

1. US1 → behaviours and developer override ship (one commit per coherent change, per constitution)
2. US2 → difficulty mechanics; rooms already default to Medium, so bots become slightly slower everywhere
3. US3 → host-only control in web and terminal host, docs
4. Polish → gates, docs, smoke
