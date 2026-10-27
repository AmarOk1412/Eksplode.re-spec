# Tasks: Player Customization, New Powers, Sound and Developer Mode

**Input**: Design documents from `/specs/003-customization-powers-audio/`

**Prerequisites**: plan.md, spec.md, research.md (R1-R16), data-model.md, contracts/ (protocol-changes, powers-rules, render-api-changes, audio-and-settings, cli-and-server-options), quickstart.md

**Tests**: Required by the constitution (principle II): every task lands with its tests, written first and seen failing (red), then green. Acceptance scenarios of this feature are named `f3_us<N>_as<M>_*` in test names (the prefix `f3_` keeps them apart from the scenarios of feature 002); scenarios that need eyes or ears are recorded in `specs/003-customization-powers-audio/checklists/manual-003.md`.

**Organization**: Tasks are grouped by user story. The phases follow the order requested for delivery (customization, powers and maluses, semi-3D lobby, noise and settings, developer mode), so the lobby part of User Story 1 is a later phase (Phase 5) that depends on Phase 3.

## Format: `- [ ] T### [P?] [Story?] Description with file path`

- **[P]**: can run in parallel (different files, no dependency on an unfinished task)
- **[Story]**: US1 customization and lobby, US2 powers, US3 sound, US4 developer mode
- Paths are relative to `game/` unless they start with `specs/`
- Commit style: `part: short desc` (at most 80 characters) plus a body; parts `core`, `bots`, `net`, `cli`, `web`, `render`, `build`, `docs`, `test`

---

## Phase 1: Setup

**Purpose**: scaffolding shared by all stories

- [X] T001 Extend `scripts/check-scenarios.sh` to take a feature prefix and id list: keep the 002 ids and add the 003 ids `f3_us1_as1`..`f3_us1_as7`, `f3_us2_as1`..`f3_us2_as9`, `f3_us3_as1`..`f3_us3_as6`, `f3_us4_as1`..`f3_us4_as6`, each satisfied by a test name in `crates/*/tests`, `crates/web/js-tests` or a line in `specs/003-customization-powers-audio/checklists/manual-003.md`; it must fail while ids are missing (red now, green at the end)
- [X] T002 [P] Create `specs/003-customization-powers-audio/checklists/manual-003.md` with empty sections for the manual checks: editor and match looks (US1 AS1, AS5), the 3D lobby compared with the reference image (US1 AS3), listening to every sound and both tunes and volume changes (US3 AS1-AS3, AS6), power icons and malus effects by eye (US2 AS9), and the dev panel (US4 AS1)
- [X] T003 [P] Add the 003 gates to `docs/DEVELOPMENT.md` (the gate list gains no new command; note the `f3_` naming rule and the manual checklist) and keep `.github/workflows/ci.yml` running `scripts/check-scenarios.sh`

---

## Phase 2: Foundational (blocking prerequisites)

**Purpose**: the web routes and module shells that US1 and US3 both need

**CRITICAL**: no user story work starts before this phase is complete

- [X] T004 Write failing route tests in `crates/web/tests/http.rs`: `GET /look.js` and `GET /audio.js` return `200 text/javascript; charset=utf-8` with the standard security headers and `HEAD` without body; `sc005_the_whole_page_stays_small` still limits the page to 500 KB and the wasm to 300 KB
- [X] T005 Add `/look.js` and `/audio.js` to `lookup` in `crates/web/src/assets.rs` (five routes become seven) and create the module shells `crates/web/assets/look.js` and `crates/web/assets/audio.js` (an exported constant each, no behaviour yet); import both from `crates/web/assets/app.js` with `import ... from "/look.js"`; update the route table in `docs/WEB.md` and `specs/002-web-client/contracts/http-and-websocket.md` (note) so T004 passes
- [X] T006 [P] Make `crates/web/js-tests/wasm-helper.mjs` expose the future exports defensively (`ex.eksplode_re_set_looks`, `ex.eksplode_re_portrait3d` optional) and add a `fakeAudio()` helper in `crates/web/js-tests/audio-helper.mjs` (fake `AudioContext` recording created nodes, scheduled notes, gain values and `suspend`/`resume` calls) for later tests

**Checkpoint**: routes and shells exist; gates green

---

## Phase 3: User Story 1 - Customize my player (Priority: P1) MVP

**Goal**: choose colour, gender look and accessories with a live preview; the host validates and de-duplicates colours; everybody sees the look in the lobby and in the match; the choice is remembered. (The semi-3D cards are Phase 5.)

**Independent Test**: open the page, change the look, join a server with bots, see the look in the lobby and the match, open a second tab and see it there, join from a terminal and see a default look.

### Tests for User Story 1 (write first, must fail)

- [X] T007 [P] [US1] Look decoding tests in `crates/net/tests/look.rs`: `Look::from_wire` turns missing, wrongly typed and unknown fields into the slot default per field and a non-object into the full default (never an error); defaults per slot are slot 0 Blue/Male, slot 1 Black/Female, slot 2 Red/Male, slot 3 Purple/Female with head `None` and face `None`; allowed values are colour `Blue, Black, Red, Purple, Green, Orange, Yellow, Pink, Teal, White`, gender `Male, Female`, head `None, Cap, TopHat, Crown, Headphones`, face `None, Glasses, Sunglasses, Moustache, Mask`; wire names are the strings; JSON round trip (`f3_us1_as7_defaults_for_bots_and_terminal_players`)
- [X] T008 [P] [US1] Host tests in `crates/net/tests/looks.rs` (sequential joins): a `Join` with a look shows it in `Welcome.looks` and in the `Lobby` message every client receives; `f3_us1_as4_a_taken_colour_moves_to_the_next_free_one_and_the_sender_is_told` (next free colour in palette order after the requested one, wrapping; earlier slots keep theirs; `Error{code:"LookAdjusted"}` names the colour); `f3_us1_as6_set_look_changes_everybody_s_lobby_and_is_refused_during_a_match` (`Error{code:"WrongPhase"}`); colours stay distinct among four participants including bots; a slot freed on leave frees its colour; a `Join` without `look` and an old-style `Welcome` consumer still work (`f3_us1_as7_`)
- [X] T009 [P] [US1] WebSocket tests in `crates/web/tests/looks.rs`: two tabs see each other's chosen look in `Lobby`; `f3_us1_as2_` is covered by the JS storage test; hostile looks (numbers, nested objects, 100 KB strings within the 64 KiB limit, unknown values) never produce `Malformed` and never affect the other player; add the new messages to the random-frame fuzz in `crates/web/tests/fuzz.rs`
- [X] T010 [P] [US1] Skin tests in `crates/web-render/tests/look.rs`: the default looks of slots 0..3 draw the same four characters as today (helmet and body colours of `player_skin`); each colour gives a different body colour; `Female` adds pixels (ponytail and lashes) that `Male` does not have; each of `Cap, TopHat, Crown, Headphones, Glasses, Sunglasses, Moustache, Mask` changes at least one pixel run versus `None`; overlays stay within sprite width plus 1 pixel per side; template rows keep their widths; `eksplode_re_set_looks` returns `0` for a JSON array, `1` for invalid UTF-8, `2` for non-array JSON, and bad entries use defaults (`crates/web-render/tests/ffi.rs`); a player drawn in a match uses the loaded look of its slot (`f3_us1_as5_`)
- [X] T011 [P] [US1] Page tests in `crates/web/js-tests/look.test.mjs`: option lists, editor reducer (set colour, gender, head, face, randomise), storage under `eksplode.re.look.v1` (save, load, corrupt JSON gives defaults, missing or throwing `localStorage` gives defaults) (`f3_us1_as2_the_look_is_remembered_after_a_reload`), preview request calls `eksplode_re_set_looks` then `eksplode_re_portrait` (`f3_us1_as1_the_preview_changes_at_once`), reducer keeps `looks` from `Welcome` and `Lobby` and shows the `LookAdjusted` message as a notice (`f3_us1_as4_`)
- [X] T012 [P] [US1] Consistency test in `crates/web/tests/consistency.rs`: the option lists exported by `crates/web/assets/look.js` (read as text) equal the lists of `net::look` and the lists in `docs/PROTOCOL.md`
- [X] T013 [P] [US1] Terminal tolerance test in `crates/cli/tests/online.rs`: an `Online` session receiving `Welcome` and `Lobby` messages that carry `looks` keeps working and shows the default tags (`f3_us1_as7_terminal_players_get_a_default_look`)

### Implementation for User Story 1

- [X] T014 [US1] Implement `crates/net/src/look.rs` (new, exported from `crates/net/src/lib.rs`): `Colour`, `Gender`, `Head`, `Face`, `Look` with the exact values of T007, `Look::default_for(slot)`, `Look::from_wire(&serde_json::Value, slot)` (lenient, never fails), `resolve_colours` (next free colour in palette order, wrapping, reports adjustments), doc comments on every public item
- [X] T015 [US1] Extend `crates/net/src/protocol.rs`: `ClientMsg::Join` gets `look: Option<serde_json::Value>` (`#[serde(default)]`), new `ClientMsg::SetLook { look }`, `ServerMsg::Welcome` and `ServerMsg::Lobby` get `looks: Vec<Look>` (`#[serde(default)]`, four entries when sent)
- [X] T016 [US1] Implement looks in `crates/net/src/host.rs`: a `looks: [Look; 4]` table filled on join (from the lenient decode), refreshed by `SetLook` in the lobby only (`Error{WrongPhase}` during a match), defaults for bots and terminal players, colour resolution on every change with `Error{LookAdjusted}` to the sender, cleared when a slot frees, `looks` in `Welcome` and every lobby broadcast; keep `Client::connect` in `crates/net/src/client.rs` sending no look
- [X] T017 [P] [US1] Implement `crates/web-render/src/look.rs` (new: `Look` skin derivation per `contracts/render-api-changes.md`: body = colour, helmet = colour mixed with white, a very dark body gets orange eyes and buckle) and layered templates in `crates/web-render/src/sprites.rs` (female ponytail and lashes overlay, head and face accessory overlays on named rows)
- [X] T018 [US1] Use the loaded looks in `crates/web-render/src/commands.rs` (`emit` for players, `portrait`) and add `eksplode_re_set_looks` to `crates/web-render/src/ffi.rs` (state holds four looks, replaced as a whole, defaults for missing); rebuild with `./scripts/build-wasm.sh`
- [X] T019 [US1] Implement `crates/web/assets/look.js`: palette, gender, head and face option lists, editor state reducer, `loadLook()` and `saveLook()` with `eksplode.re.look.v1` and safe fallbacks, helpers to build the `Join` and `SetLook` payloads
- [X] T020 [US1] Page integration in `crates/web/assets/index.html`, `crates/web/assets/style.css` and `crates/web/assets/app.js`: the character editor on the join screen (colour swatches, gender toggle, head and face pickers, live preview canvas through `renderer.setLooks` and `renderer.portrait`), a "Change look" button in the lobby that sends `SetLook` (disabled during a match), `look` in `Join`, `looks` kept from `Welcome` and `Lobby` and pushed to the module, the `LookAdjusted` notice, wrapper methods `setLooks` in `createRenderer`
- [X] T021 [P] [US1] Update `docs/PROTOCOL.md` (Join.look, SetLook, Welcome and Lobby looks, `LookAdjusted`, look wire table) and `docs/WEB.md` (editor, storage key) so the docs tests list every new variant
- [X] T022 [US1] Extend `scripts/browser-smoke.mjs` to pick a look, join, save screenshots of the editor, the lobby and the match, and print the stored look; run T007-T013, the gates and the smoke script; tick the US1 manual items in `specs/003-customization-powers-audio/checklists/manual-003.md`. Commits: `net: add looks with colour resolution and SetLook`, `render: draw players from their look`, `web: add the character editor`

**Checkpoint**: customization works end to end; MVP

---

## Phase 4: User Story 2 - Play with a richer set of powers, including the punch (Priority: P2)

**Goal**: roller, flame and extra bomb, the rare punch, and the maluses slow, hyper speed and small bomb, with icons, drops decided at block destruction, the same rules in the terminal and in the browser.

**Independent Test**: with a fixed seed or a prepared arena, pick up each power and check speed, range, bomb reach and punching; punch a bomb over blocks and players and check where it lands.

### Tests for User Story 2 (write first, must fail)

- [X] T023 [P] [US2] Drop table tests in `crates/core/tests/drops.rs`: seven chances in `PowerUpKind::ALL` order `ExtraBomb, Range, Speed, Punch, Slow, HyperSpeed, SmallBomb`, each `0.0..=1.0` and finite (validation errors otherwise); defaults equal `powerup_chance` times the shares .20, .24, .24, .03, .10, .08, .11 and sum to `powerup_chance`; `f3_us2_as9_` rarity: over 1000 destroyed blocks the share of every kind is within 3 percentage points of its default and `Punch` is at most 5% of drops; with `Punch` at 1.0 and the others at 0.0 every destroyed block drops `Punch` and with all zeros none drops (200 draws); sums above 1.0 are scaled by `1 / sum`; at most one item per block; same seed and inputs give the same items
- [X] T024 [P] [US2] Pickup and malus tests in `crates/core/tests/malus.rs`: `f3_us2_as1_` pickup removes the item, applies at once and emits `PowerUpCollected`; `f3_us2_as2_` roller lowers the move cooldown by one down to `min_move_cooldown` and flame raises range by one up to `max_range`; `f3_us2_as6_` slow gives effective cooldown `ceil(base * 4 / 3)` and ends after exactly `malus_ticks` (10 seconds, 200 ticks at 20 Hz) with `MalusEnded`; `f3_us2_as7_` hyper speed gives `max(1, base / 2)` and ends the same way; `f3_us2_as8_` small bomb gives bombs placed during it range 1 while earlier bombs keep theirs; a new malus replaces the old one, the same malus restarts the timer, elimination and a new round clear it, bonuses survive a malus
- [X] T025 [P] [US2] Punch tests in `crates/core/tests/punch.rs`: `f3_us2_as3_` free next square, one block, two blocks, wall, player, another bomb, mixed obstacles, each landing on the first square without block, wall, living player or bomb; a player stepping onto the landing square during the flight moves the landing on; `f3_us2_as4_` edge of the arena lands on the nearest free square back toward the origin (the origin counts); fuse keeps burning and a bomb whose fuse ends in flight explodes at its current square; flames do not trigger a bomb in flight; chain reaction after landing; `f3_us2_as5_` punch without the power, without a bomb in front, or on a bomb already in flight does nothing; hops take `punch_hop_ticks` (2) ticks each and emit `BombPunched`, `BombHopped`, `BombLanded`; 20 prepared situations in one table test
- [X] T026 [P] [US2] Update existing tests that rely on `hidden_powerups` (`crates/core/tests/powerups.rs`, `crates/core/tests/determinism.rs`, `crates/core/tests/chain.rs`, `crates/web-render/tests/fixtures.rs` and the fixture `crates/web-render/tests/fixtures/observation.json`) to the destruction-time drop model, config tests for `malus_ticks` and `punch_hop_ticks` (default 10 s times `tick_hz` and 2, validation "at least 1", old `Config` JSON without them still deserializes) in `crates/core/src/config.rs`, and a whole-match determinism test with the new rules
- [X] T027 [P] [US2] Bot tests in `crates/bots/tests/danger.rs` and `crates/bots/tests/behaviour.rs`: a bomb in flight counts as dangerous at its landing prediction and at its current square, bots walk to bonuses and avoid maluses, no bot gets stuck or panics on any new power, `crates/bots/tests/soak.rs` still finishes 100 four-bot matches
- [X] T028 [P] [US2] Protocol tests in `crates/net/tests/loopback.rs`: `Input` without `punch` means false, `Input.punch` reaches `step` (a player with the punch punches through the host), observation JSON carries `has_punch`, `malus` and bomb `flight`
- [X] T029 [P] [US2] Terminal tests in `crates/cli/tests/keys.rs` and `crates/cli/tests/view.rs`: punch keys P1 `z`, P2 `c`, P3 `u`, P4 `n` do not clash with `q`, `p`, `r`, `b`, `x` or any movement and bomb key; power letters `B` extra bomb, `R` roller, `F` flame, `P` punch, `S` slow, `H` hyper speed, `s` small bomb (bonuses green, maluses red); a bomb in flight is `()` with a trailing `~`; status lines show `punch`, `slow 7s`, `hyper 4s`, `small 9s` (`f3_us2_as9_` for the terminal)
- [X] T030 [P] [US2] Render tests in `crates/web-render/tests/commands.rs`: the seven power icons are distinct per kind, bonuses have a white frame and maluses a dark red frame, a bomb in flight is lifted by `sin(pi * progress) * 0.5` tile with `progress = 1 - hop_left / punch_hop_ticks` and has a shadow, a player with a malus gets the coloured pulse (slow blue, hyper yellow, small bomb red), stride stays 16; node check in `crates/web/js-tests/render.test.mjs`
- [X] T031 [P] [US2] Page tests in `crates/web/js-tests/input.test.mjs` and `crates/web/js-tests/app.test.mjs`: `KeyTracker` sets `punch` for `KeyZ` and `KeyQ` and clears it on release and blur, `inputMessage` carries `punch`, `hudRows` lists active effects with seconds left and a punch mark

### Implementation for User Story 2

- [X] T032 [US2] Core data in `crates/core/src/state.rs`, `crates/core/src/action.rs`, `crates/core/src/config.rs`: `PowerUpKind` gains `Punch, Slow, HyperSpeed, SmallBomb` with `ALL` in the order of T023 and `is_malus()`; `Participant` gains `has_punch` and `malus: Option<ActiveMalus { kind, ticks_left }>`; `Bomb` gains `flight: Option<Flight { dir, hop_left }>`; `PlayerInput` gains `punch: bool` (`#[serde(default)]`); `Config` gains `malus_ticks` and `punch_hop_ticks` with serde defaults and validation; new `Event` variants `BombPunched`, `BombHopped`, `BombLanded`, `MalusStarted`, `MalusEnded`; doc comments on all public items
- [X] T033 [US2] Implement `crates/core/src/drops.rs` (new): `DropTable` with `defaults`, `validate`, scaling, `roll`; replace `hidden_powerups` in `GameState` by `drops`, remove the hiding loop in `GameState::new`, decide items per destroyed block in `crates/core/src/bomb.rs` using the state RNG in position order
- [X] T034 [US2] Implement `crates/core/src/punch.rs` (new) and the step order of `contracts/powers-rules.md` in `crates/core/src/step.rs` and `crates/core/src/bomb.rs`: malus timers, pickups for the new kinds (replace/restart rules), punch before placement, flights advance before detonation, flying bombs do not block movement or placement and are skipped by flame triggering, small-bomb range on placement, malus cleared on elimination; make T023-T026 pass
- [X] T035 [US2] Bots in `crates/bots/src/danger.rs` and `crates/bots/src/default_bot.rs`: flights in the danger map using the same landing prediction (`core::punch::predict_landing`), scoring weights for bonuses and maluses; bots never set `punch`; make T027 pass
- [X] T036 [US2] Net in `crates/net/src/protocol.rs` and `crates/net/src/host.rs`: `Input.punch` (`#[serde(default)]`) passed into `PlayerInput`; make T028 pass
- [X] T037 [US2] Terminal in `crates/cli/src/input.rs`, `crates/cli/src/view.rs`, `crates/cli/src/app.rs` and `crates/cli/src/online.rs`: punch keys per player (local session and online `Online::send_input`), power letters and colours, flight marker, effect list in the status line; make T029 pass
- [X] T038 [P] [US2] Render in `crates/web-render/src/sprites.rs` and `crates/web-render/src/commands.rs`: pixel templates for the seven icons (letter fallback below a small scale), frames, flight arc, malus pulse; regenerate `crates/web-render/tests/fixtures/observation.json` and rebuild the wasm; make T030 pass
- [X] T039 [US2] Page in `crates/web/assets/app.js`, `crates/web/assets/index.html`, `crates/web/assets/style.css`: punch keys (`KeyZ`, `KeyQ`) in `KeyTracker` and `inputMessage`, HUD effects (icons and seconds left), controls hint with the punch key; make T031 pass
- [X] T040 [P] [US2] Documentation: new `docs/GAMEPLAY.md` (powers table with chances, malus numbers, punch rules and edge cases, controls for terminal and browser), README controls table gains the punch keys and links to GAMEPLAY, `docs/PROTOCOL.md` (`Input.punch`, observation fields, events), `docs/ARCHITECTURE.md` step order
- [X] T041 [US2] Run T023-T031, the gates and `cargo test -p eksplode-re-bots --test soak -- --nocapture`; play a round in the browser and in the terminal with a fixed seed and record the by-eye checks (icons, hops, malus pulse) in `specs/003-customization-powers-audio/checklists/manual-003.md`. Commits: `core: add drop table, maluses and the punch`, `bots: handle bombs in flight and new powers`, `cli: punch keys, power letters and effect status`, `render: power icons, bomb flight and malus pulse`, `web: punch keys and effects in the status line`

**Checkpoint**: all seven powers work in core, bots, terminal and browser

---

## Phase 5: User Story 1 (continued) - Semi-3D lobby (Priority: P1)

**Goal**: lobby cards drawn as isometric cubic characters after the reference image (tall card, large cubic character, name banner, player name below, empty cards, owner/you/ready marks).

**Independent Test**: open the lobby with four participants and compare it with the reference image.

**Depends on**: Phase 3 (looks).

### Tests (write first, must fail)

- [X] T042 [P] [US1] 3D tests in `crates/web-render/tests/iso3d.rs`: projection is 2:1 and round-trips; `eksplode_re_portrait3d` returns commands only inside the given rectangle for several sizes, returns 0 for invalid sizes (0, negative, NaN), is deterministic for the same `phase`, differs per slot, per colour and per accessory, head faces are shaded 0.8 left, 0.6 right, 1.0 top, the pixel face is drawn on the front-right face as skewed quads, the vertical bob is at most 4% of the height with a 2 s period and the yaw swap happens every 6 s (`f3_us1_as3_`); commands are ordered back to front; the wasm export exists (`crates/web-render/tests/ffi.rs`)
- [X] T043 [P] [US1] Page tests in `crates/web/js-tests/lobby.test.mjs`: the card model for a lobby has one card per slot with banner text `PLAYER n`, the player name caption (`name (you, owner, ready)` tags as today), empty slots as empty cards, bots labelled `bot`; the animation loop runs only while the lobby is visible (`f3_us1_as3_each_occupied_slot_is_a_card_with_banner_and_name`); the node render test calls `eksplode_re_portrait3d` through the glue (`crates/web/js-tests/glue.test.mjs`)

### Implementation

- [X] T044 [US1] Implement `crates/web-render/src/iso3d.rs` (new: isometric projection, cube faces, body and head cubes, arms and feet, pixel face on the front-right face, accessory boxes, ground shadow, bob and yaw swap) and export it from `crates/web-render/src/lib.rs`
- [X] T045 [US1] Add `eksplode_re_portrait3d(slot, width, height, phase)` to `crates/web-render/src/ffi.rs` and `portrait3d` to `createRenderer` in `crates/web/assets/app.js`; rebuild the wasm and check the size stays at most 300 KB
- [X] T046 [US1] Redesign the lobby in `crates/web/assets/app.js`, `crates/web/assets/index.html`, `crates/web/assets/style.css`: tall rounded cards with the 3D canvas, a coloured name banner (`PLAYER n`), the player name below, empty cards, owner/you/ready marks, a `requestAnimationFrame` loop that repaints only visible lobby cards and stops outside the lobby, readable with very long names (ellipsis); keep the menu logo on the flat `eksplode_re_portrait`
- [X] T047 [US1] Capture the lobby with `scripts/browser-smoke.mjs` (four participants), compare with the reference image, tune proportions and colours, record the comparison in `specs/003-customization-powers-audio/checklists/manual-003.md`; update the lobby paragraphs in `README.md` and `docs/WEB.md`. Commits: `render: add the semi-3D character for lobby cards`, `web: show the lobby as 3D character cards`

**Checkpoint**: US1 complete

---

## Phase 6: User Story 3 - Hear the game and set the volume (Priority: P3)

**Goal**: generated sound effects and two looping tunes, volume and mute settings that persist; nothing breaks without sound.

**Independent Test**: press a key, hear lobby and match tunes and the effects, change volumes and mute, reload.

### Tests (write first, must fail)

- [X] T048 [P] [US3] Lifecycle and settings tests in `crates/web/js-tests/audio.test.mjs` using `fakeAudio()`: no context is created before the first `keydown`, `pointerdown` or `touchstart` (`f3_us3_as5_`), missing or throwing `AudioContext` makes every call a no-op, three buses (`master`, `music`, `effects`), effective gain `muted ? 0 : value` with a 50 ms ramp (`f3_us3_as3_volume_changes_apply_at_once`), settings `{ music: 0..1, effects: 0..1, muted: bool }` clamp, defaults `0.35 / 0.7 / false`, stored under `eksplode.re.audio.v1`, corrupt or unavailable storage gives defaults (`f3_us3_as4_settings_survive_a_reload`), `document.hidden` suspends and visible resumes (`f3_us3_as6_`)
- [X] T049 [P] [US3] Sound tests in `crates/web/js-tests/sounds.test.mjs`: the event-to-sound map (`BombPlaced` bombPlace, `BombExploded` explosion, `PowerUpCollected` bonus or malus by kind, `BombPunched` punch, `BombLanded` land, `PlayerEliminated` eliminated, `RoundEnded` win or draw, button clicks `click`), at most one sound per id per snapshot and at most 8 starts per snapshot, each recipe schedules oscillators or noise with the durations of `contracts/audio-and-settings.md` (`f3_us3_as2_every_listed_event_has_its_own_sound`)
- [X] T050 [P] [US3] Music tests in `crates/web/js-tests/music.test.mjs`: the sequencer schedules the expected note list for a fixed horizon (25 ms timer, 120 ms lookahead), the `lobby` tune (96 BPM, C major pentatonic arpeggio over a 4-bar bass) and the `match` tune (144 BPM, A natural minor, driving 8th-note bass, 2-bar melody, noise hi-hat) are identical on every run (fixed seeds) and loop, switching tunes fades 300 ms, the lobby tune plays in the lobby and on the join screen and the match tune during a match (`f3_us3_as1_`)
- [X] T051 [P] [US3] Docs consistency test in `crates/web/tests/docs.rs`: every sound id and tune name found in `crates/web/assets/audio.js` appears in `docs/AUDIO.md` together with the words "generated" and its recipe, and no audio file extension (`.mp3`, `.ogg`, `.wav`, `.flac`) is referenced or embedded (`crates/web/src/assets.rs`) (SC-009)

### Implementation

- [X] T052 [US3] Implement the synthesiser and settings in `crates/web/assets/audio.js`: lazy context on first gesture, buses and ramps, settings load/save, hidden-page suspend/resume, safe no-ops
- [X] T053 [US3] Implement the sound effects and the event mapping in `crates/web/assets/audio.js` (recipes of `contracts/audio-and-settings.md`: oscillators with envelopes and filtered noise, explosion volume scaled for several per tick) and call it from the `Snapshot` handler of `crates/web/assets/app.js`; make T049 pass
- [X] T054 [US3] Implement the lookahead sequencer and both tunes in `crates/web/assets/audio.js` (patterns from a scale and a small seeded generator with fixed seeds) and switch tunes with the page stage in `crates/web/assets/app.js`; make T050 pass
- [X] T055 [US3] Settings panel in `crates/web/assets/index.html`, `crates/web/assets/style.css`, `crates/web/assets/app.js`: a speaker button on the join screen, lobby and match opens the panel with a music slider, an effects slider, a mute toggle and a "test sound" button; sliders apply immediately and persist; `click` sound on buttons
- [X] T056 [P] [US3] Write `docs/AUDIO.md` (every sound and tune with its recipe, that all audio is generated at run time, no samples, browser rules, settings) and link it from `README.md` and `docs/WEB.md`; make T051 pass
- [X] T057 [US3] Run T048-T051, the gates and the CSP check (no inline script, no media requests); listen to every sound and both tunes, test volume changes within 0.2 s, mute, reload, a blocked-audio browser and a hidden tab, and record the results in `specs/003-customization-powers-audio/checklists/manual-003.md`. Commits: `web: add generated sound effects and music`, `web: add the sound settings panel`, `docs: describe how the audio is generated`

**Checkpoint**: US3 complete

---

## Phase 7: User Story 4 - Developer mode (Priority: P4)

**Goal**: `--dev` enables owner-only commands to pause bots and set the drop chance of each of the seven powers; everything is refused without `--dev`.

**Independent Test**: start with `--dev`, pause bots, set one power to 100%, then start a normal server and check the refusals.

**Depends on**: Phase 4 (drop table).

### Tests (write first, must fail)

- [X] T058 [P] [US4] Host tests in `crates/net/tests/dev.rs`: `f3_us4_as5_` every `Dev*` message gets `Error{DevDisabled}` and changes nothing when the host was not started in developer mode; with it on, a non-owner gets `Error{NotOwner}`; `f3_us4_as2_` `DevPauseBots { paused: true }` makes every bot give empty input within one second (20 ticks) while placed bombs keep burning and human inputs are unaffected, and `paused: false` resumes them; `f3_us4_as3_` `DevSetDrops` with `Punch` 1.0 and the rest 0.0 makes every destroyed block drop `Punch` and 0.0 for `Punch` never drops it; `f3_us4_as4_` the new table applies mid-match to blocks destroyed afterwards and `DevResetDrops` restores the defaults; chances outside `0.0..=1.0` or not finite give `Error{BadValue}` and no change, sums above 1.0 are scaled; `DevState { paused, drops }` is broadcast on change and sent to a joiner, `Welcome.dev` is true only in developer mode
- [X] T059 [P] [US4] Server tests in `crates/web/tests/dev.rs` and `crates/web/tests/args.rs`: `--dev` parses (alone and with other options) and is off by default; the start-up warning line starts with `WARNING:` and says developer mode is for testing and not for untrusted networks; the banner shows `Developer mode: on`; the same flow works over WebSocket (`f3_us4_as1_the_dev_panel_is_offered_only_in_dev_mode` through `Welcome.dev`)
- [X] T060 [P] [US4] CLI tests in `crates/cli/tests/args.rs` and `crates/cli/tests/online.rs`: `host --dev` parses and `host_warnings` returns the developer warning (`f3_us4_as6_`), default `host` does not
- [X] T061 [P] [US4] Page tests in `crates/web/js-tests/dev.test.mjs`: the reducer keeps `dev` and `devState`, the panel model offers pause/resume, seven sliders 0-100% in power order and reset only when `dev` is true, a "DEV MODE" marker is shown, slider values map to chances `0.0..=1.0`, only the owner can use it

### Implementation

- [X] T062 [US4] Protocol in `crates/net/src/protocol.rs`: `ClientMsg::DevPauseBots { paused }`, `ClientMsg::DevSetDrops { chances }` (seven `f32` in order `ExtraBomb, Range, Speed, Punch, Slow, HyperSpeed, SmallBomb`, each finite and `0.0..=1.0`), `ClientMsg::DevResetDrops`, `ServerMsg::DevState { paused, drops }`, `Welcome.dev: bool` (`#[serde(default)]`); new error codes `DevDisabled` and `BadValue`
- [X] T063 [US4] Host in `crates/net/src/host.rs` and `HostConfig`: `dev: bool`, `bots_paused`, the live `DropTable` applied to the running `GameState` between ticks through a `set_drops` method in `crates/core/src/state.rs`, bot decisions skipped while paused, owner and enabled checks, `DevState` broadcasts; make T058 pass
- [X] T064 [US4] Server in `crates/web/src/args.rs`, `crates/web/src/options.rs`, `crates/web/src/lib.rs`, `crates/web/src/main.rs`, `crates/web/src/origin.rs`/`addr.rs`: `--dev` option and `USAGE` line (`--dev  developer mode: owner-only commands to pause bots and set drop chances (testing only)`), warning, banner line; regenerate the options table in `docs/WEB.md`; make T059 pass
- [X] T065 [US4] CLI in `crates/cli/src/args.rs`, `crates/cli/src/app.rs`: `host --dev` and its `USAGE` text, `host_warnings` developer line, `HostConfig.dev`; make T060 pass
- [X] T066 [US4] Dev panel in `crates/web/assets/index.html`, `crates/web/assets/style.css`, `crates/web/assets/app.js`: visible only when `Welcome.dev`, marker "DEV MODE", pause/resume bots, seven drop sliders 0-100% with the power names, reset button, state follows `DevState`; make T061 pass
- [X] T067 [P] [US4] Documentation: developer mode section in `docs/WEB.md` and `docs/SECURITY.md` (testing only, never on an exposed network), `docs/PROTOCOL.md` (`Dev*` messages, `DevState`, `DevDisabled`, `BadValue`), README option list (`--dev`), `docs/GAMEPLAY.md` (how to force a power)
- [X] T068 [US4] Run T058-T061, the gates and the smoke script against `--dev` and a normal server (panel shown only in dev, commands refused otherwise, bots stop within 1 s); tick the US4 manual items in `specs/003-customization-powers-audio/checklists/manual-003.md`. Commits: `net: add developer mode with bot pause and live drop table`, `web: add the developer panel and --dev`, `cli: add host --dev`

**Checkpoint**: all four stories complete

---

## Phase 8: Polish & Cross-Cutting Concerns

- [X] T069 [P] Audit non-test code in `crates/net/src/look.rs`, `crates/net/src/host.rs`, `crates/core/src/punch.rs`, `crates/core/src/drops.rs` and `crates/web-render/src/{look,iso3d,ffi}.rs` for `unwrap`, `expect`, `panic!`, unchecked indexing and unchecked arithmetic on external data (look values, chances, wasm lengths, hop counters) and replace with typed errors; extend `crates/web/tests/fuzz.rs` and `crates/web-render/tests/ffi.rs` with random bytes for `eksplode_re_set_looks` and `eksplode_re_portrait3d` and random frames for every new message
- [X] T070 [P] Performance and size checks recorded in `docs/DEVELOPMENT.md` and `specs/003-customization-powers-audio/checklists/manual-003.md`: key-to-snapshot p95 at most 200 ms with the new rules (`cargo test -p eksplode-re-web --test mixed sc003 -- --nocapture`), wasm at most 300 KB, page at most 500 KB, 60 fps in the animated lobby and in a match by eye (SC-010)
- [X] T071 [P] Add a mixed-match test in `crates/web/tests/mixed.rs`: a browser, a terminal client and a bot share a match with new powers on the board; all see the same looks, the same item drops (same seed) and the same result (`f3_us2_as9_powers_work_the_same_for_every_client`)
- [X] T072 Finish `scripts/check-scenarios.sh` and `specs/003-customization-powers-audio/checklists/manual-003.md` so all 28 scenario ids (`f3_us1_as1`..`f3_us4_as6`) are covered; the script must pass (T001 turns green)
- [X] T073 [P] Documentation sweep: README (features, controls with the punch key, options with `--dev`), `docs/ARCHITECTURE.md` (new modules `look`, `drops`, `punch`, `iso3d`, `audio.js`; crate sections; diagram note), `docs/DEVELOPMENT.md` (how to add a power, a look option, a sound), `docs/SECURITY.md`; run `cargo test -p eksplode-re-web --test docs`
- [X] T074 Update `specs/002-web-client/contracts/wasm-render-api.md` and `specs/002-web-client/contracts/http-and-websocket.md` with a note pointing to the 003 contracts (new exports, new routes) and keep `specs/003-customization-powers-audio/contracts/` as the source for the changes
- [X] T075 Run the full gates (`cargo fmt --check`, `cargo clippy --workspace --all-targets -- -D warnings`, `cargo test --workspace`, `./scripts/build-wasm.sh`, `(cd crates/web/js-tests && node --test)`, `python3 scripts/pty-enter-check.py --attempts 20`, `scripts/check-scenarios.sh`), the whole `quickstart.md`, and a clean-clone run of the README quick start; fix gaps
- [X] T076 Review the history of `game/` against the commit convention (`git log --format=%s | awk 'length>80'` is empty and every subject matches the hook) and record the success-criteria results SC-001..SC-012 in `specs/003-customization-powers-audio/checklists/manual-003.md`. Commit: `docs: finalise customization, powers and audio notes`

---

## Phase 9: Amendment A1 - pixel-art icons, animated directional sprites, kick

**Goal**: FR-011 (pixel-art icons), FR-034/FR-035 (views and walk cycle), FR-032/FR-033 (kick), scenarios `f3_us1_as8`, `f3_us1_as9`, `f3_us2_as10`, `f3_us2_as11`, `f3_us2_as12`, SC-013, SC-014. The bonus requested as "roller" is called **kick** (see Assumptions).

**Independent Test**: play a round with the dev table set to 100% kick: the boot icon appears, walking into a bomb slides it to the next blocker, players walk with a visible step animation and turn to every direction.

### Tests (write first, must fail)

- [X] T077 [P] [US2] Core tests in `crates/core/tests/kick.rs` (new) and `crates/core/tests/drops.rs`: `f3_us2_as10_`/`as11_` kick cases (run to a wall, a block, another bomb, a living player, the edge, no room behind, no kick, flying bomb not kickable, punching a sliding bomb, fuse ends while sliding, flames on a sliding bomb, a player stepping into the path, step timing of `slide_ticks`, events `BombKicked`/`BombSlid`/`BombStopped`, pickup sets `has_kick`, round start clears it, determinism); the drop table has eight chances with `Kick` last and the new default shares; update every test that builds a table or counts kinds
- [X] T078 [P] [US2] Bots, net and cli tests: `crates/bots/tests/danger.rs` (`f3_us2_as12_` a sliding bomb is dangerous at its square and at its stop square, via `core::kick::predict_slide_stop`), `crates/net/tests/dev.rs` and `codec.rs` (eight chances, seven refused with `BadValue`, `Kick` at 100% makes every block drop a kick), `crates/web/tests/dev.rs`/`fuzz.rs` as needed, `crates/cli/tests/view.rs` (`+K` green, a sliding bomb drawn `(>` `(<` `(^` `(v`, status line shows `kick`)
- [X] T079 [P] [US2] Render tests in `crates/web-render/tests/commands.rs` and `tests/icons.rs` (new): `f3_us2_as10_` every kind has its own pixel template (distinct grids, 10x10, no letter command at normal size, a letter only below 20 px), frames white for bonuses and dark red for maluses, a sliding bomb moves between squares with the same look as a landed one, stride 16
- [X] T080 [P] [US1] Render tests in `crates/web-render/tests/sprites3.rs` (new): `f3_us1_as8_` views per direction (back view has no eye pixels and no face accessory, side view has one eye, right is the mirror of left, hats and headphones differ per view), `f3_us1_as9_` walking frames alternate with distance, idle when standing, the sprite faces the last direction, a new round faces down, looks stay intact in every view (colour, gender, accessories), nothing leaves the canvas
- [X] T081 [P] [US3] Page tests: `crates/web/js-tests/dev.test.mjs` (eight sliders in order with `Kick` last, label "Kick"), `sounds.test.mjs` (`BombKicked` maps to `kick` with its recipe), `render.test.mjs` (kick icon and a walking frame through the wasm)

### Implementation

- [X] T082 [US2] Core in `crates/core/src/{state,kick,step,bomb,config,drops,lib}.rs`: `PowerUpKind::Kick` appended to `ALL`, `Participant.has_kick`, `Bomb.slide`, `kick.rs` (start, step, stop, `predict_slide_stop`), `slide_ticks` in `Config` (default 3, serde default, validated), events, step order of `contracts/powers-rules.md`, `DropTable` with eight chances and shares .19 .22 .22 .03 .10 .08 .11 .05; make T077 pass
- [X] T083 [US2] Bots in `crates/bots/src/{danger,pathing}.rs`: sliding bombs in the danger map and as obstacles; bots never kick on purpose; make the T078 bot tests pass and keep the soak test green
- [X] T084 [US2] Net in `crates/net/src/{protocol,host}.rs`: `DevState.drops` and `DevSetDrops` with eight chances (anything else is `BadValue`); observation fields come from core; make the T078 net tests pass
- [X] T085 [US2] Terminal in `crates/cli/src/view.rs`: `+K`, slide glyphs, `kick` in the status line; make the T078 cli tests pass
- [X] T086 [US2] Render icons in `crates/web-render/src/{sprites,commands}.rs`: ten-by-ten pixel templates for the eight powers, letter fallback below 20 px, kick icon, sliding bombs; regenerate `tests/fixtures/observation.json`; make T079 pass
- [X] T087 [US1] Render sprite views and animation in `crates/web-render/src/{look,commands,interp,depth}.rs`: `grid(look, view, frame)`, view and frame from the last facing and the walked distance (see render contract A1), mirrored right view, bob; make T080 pass; rebuild the wasm and keep it under 300 KB
- [X] T088 [US3/US4] Page in `crates/web/assets/{app.js,audio.js}`: eighth dev slider ("Kick"), `kick` sound and the `BombKicked` mapping; make T081 pass
- [X] T089 [P] Documentation and scenarios: `docs/GAMEPLAY.md` (kick rules, table with `Kick` and new chances, icons, sprite views), `docs/AUDIO.md` (`kick` row), `docs/PROTOCOL.md` (eight chances, observation fields), `docs/ARCHITECTURE.md`, `docs/DEVELOPMENT.md`, README; add `f3_us1_as8 f3_us1_as9 f3_us2_as10 f3_us2_as11 f3_us2_as12` to `scripts/check-scenarios.sh` and the manual checklist
- [X] T090 Run the full gates, the smoke script against a `--dev` server with the kick at 100% (screenshots of the icons and the walking sprites in four directions), record the results and SC-013/SC-014 in `checklists/manual-003.md`. Commits per part (`core: add the kick bonus`, `bots: ...`, `net: ...`, `cli: ...`, `render: draw pixel-art power icons`, `render: add directional walking sprites`, `web: ...`, `docs: ...`)

---

## Dependencies & Execution Order

- **Phase 1** then **Phase 2** (blocking: routes for `look.js` and `audio.js`).
- **Phase 3 (US1 customization)** after Phase 2; it is the MVP.
- **Phase 4 (US2 powers)** after Phase 2; independent of Phase 3 in behaviour, but both edit `crates/web-render/src/sprites.rs`, `commands.rs`, `ffi.rs` and `crates/web/assets/app.js`, so run them one after the other (Phase 3 first, as requested).
- **Phase 5 (US1 lobby)** after Phase 3 (needs looks); it can run before or after Phase 4.
- **Phase 6 (US3 audio)** after Phase 2; the sounds for `BombPunched`, `BombLanded` and malus pickups need Phase 4 events, so run it after Phase 4 (as requested).
- **Phase 7 (US4 dev mode)** after Phase 4 (drop table) and uses the panel patterns of Phases 3 and 6.
- **Phase 8** last.

Within a phase: tests (red) first, then core or net, then render, then page, then docs, then the verification task. Tasks marked [P] touch different files and can be done together.

### Parallel examples

- **US1 tests**: T007, T008, T009, T010, T011, T012, T013 together; then T014-T016 (net) in order while T017 (render skin) runs in parallel.
- **US2 tests**: T023-T031 together; implementation T032-T034 in order, then T035, T036, T037, T038 in parallel, then T039.
- **US3 tests**: T048, T049, T050, T051 together; T052 before T053 and T054; T056 in parallel with T055.
- **US4 tests**: T058, T059, T060, T061 together; T062 before T063-T066.
- **Polish**: T069, T070, T071, T073 together.

## Implementation Strategy

1. **MVP**: Phases 1-3 (customization) is a complete, demonstrable increment: looks chosen, validated, shared and drawn in the match.
2. **Then in the requested order**: Phase 4 (powers and maluses, including the punch), Phase 5 (semi-3D lobby), Phase 6 (noise, music, settings), Phase 7 (developer mode), Phase 8 (polish).
3. After each phase run the common gates and commit with the `part: desc` style; never start the next phase with a red gate.
4. Keep the protocol additive (version 1), the rules in `core`, and the page free of rules; every task lands with its tests.

## Notes

- 90 tasks (76 first delivery + 14 of Amendment A1, T077-T090): Setup 3, Foundational 3, US1 customization 16 (T007-T022), US2 19 (T023-T041), US1 lobby 6 (T042-T047), US3 10 (T048-T057), US4 11 (T058-T068), Polish 8 (T069-T076).
- Scenario naming: `f3_us<N>_as<M>_` in test names; manual scenarios live in `specs/003-customization-powers-audio/checklists/manual-003.md`.
- Known risk: moving item decisions to block destruction changes which items appear for existing seeds; T026 updates every affected test in the same change.
