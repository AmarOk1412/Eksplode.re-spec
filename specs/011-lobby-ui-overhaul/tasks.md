---
description: "Executable tasks for the menu/lobby/customize/scoreboard reorganization"
---

# Tasks: Menu, Lobby, Customization and Scoreboard Reorganization

**Input**: Design documents from `specs/011-lobby-ui-overhaul/`

**Prerequisites**: [plan.md](plan.md), [spec.md](spec.md), [research.md](research.md), [data-model.md](data-model.md), [contracts/protocol-changes.md](contracts/protocol-changes.md), [contracts/ui-screens.md](contracts/ui-screens.md), [quickstart.md](quickstart.md)

**Tests**: Required by Constitution II ("Tests MUST be written before implementation and observed failing... Every spec acceptance scenario MUST map to at least one automated test"). Every story below lists its tests first; an implementation task is not done until its own tests, and any existing test it would otherwise break, pass.

**Organization**: Six independently testable user-story increments, in spec priority order. US4 additionally depends on US3's tab container existing (noted in its phase). All other stories are mutually independent.

## Format: `[ID] [P?] [Story] Description`

- `[P]`: different files, no dependency on an incomplete task in the same phase.
- `[US1]`..`[US6]` map to the six user stories in spec.md. Setup and Polish carry no story label.
- All paths are relative to `game/` (the Cargo workspace root), e.g. `crates/core/src/lobby.rs`.

## Path Conventions

Rust workspace: `game/`; game rules: `game/crates/core/`; networking/protocol: `game/crates/net/`; WASM renderer: `game/crates/web-render/`; web service + browser assets: `game/crates/web/` (JS/HTML/CSS under `game/crates/web/assets/`, JS tests under `game/crates/web/js-tests/`). No new crates or dependencies (plan.md).

---

## Phase 1: Setup

- [X] T001 Record current baseline results for `cargo fmt --check`, `cargo clippy --all-targets -- -D warnings`, `cargo test --workspace`, and the existing Node-based suite under `crates/web/js-tests/` (see quickstart.md "Gates before merge"), so later checkpoints can tell a pre-existing failure from a regression introduced by this feature. *(baseline, with 010-player-collision already in the tree: fmt clean, clippy clean, `cargo test --workspace` 762 passed / 0 failed, `node --test` in `crates/web/js-tests/` 175 passed / 0 failed.)*

**Checkpoint**: Baseline recorded; no code changed yet.

## Phase 2: Foundational

No cross-story blocking prerequisites: each story below reads/writes a disjoint slice of `core`/`net`/`web` (US2 touches `Lobby`/bot slots, US5 touches `Player` match statistics, US4 touches `Look`, US1/US3/US6 are web-only navigation/CSS). See "Implementation Strategy" at the end for suggested sequencing even though no story *requires* another to compile or pass its own tests.

**Checkpoint**: None required — proceed directly to Phase 3.

---

## Phase 3: User Story 1 - Navigate from a Clear Main Menu (Priority: P1)

**Goal**: Replace the current direct-to-room-creation entry with a Main Menu offering Create new game / Join a game / Options.

**Independent Test**: Launch the client fresh; confirm exactly three main-menu actions and that no room-creation form shows until "Create new game" is chosen (spec Acceptance Scenarios 1-5).

### Tests for User Story 1

- [X] T002 [P] [US1] Add failing test in `crates/web/js-tests/app.test.mjs` asserting the client's initial render state is a main menu with exactly three actions ("Create new game", "Join a game", "Options") and that the create-room form is not shown until "Create new game" is activated (AS-1, AS-2). *(landed as `f11_us1_as1_the_client_opens_on_a_main_menu_with_exactly_three_actions` and `f11_us1_as2_create_new_game_opens_the_existing_room_creation_form` (plus `f11_us1_a_lost_connection_shows_the_main_menu_with_retry`); to make the menu flow testable without a DOM, panel visibility was extracted from `updateUi()` into a pure exported `screens(state)` and the actions into `MAIN_MENU_ACTIONS`. Failed first on the missing exports.)*
- [X] T003 [P] [US1] Add failing test in `crates/web/js-tests/room.test.mjs` asserting "Join a game" renders the existing room list (directory panel) with a Refresh action and a New-room action, and that Refresh re-issues `ListRooms` without changing the active screen (AS-3, AS-4). *(landed as `f11_us1_as3_join_a_game_shows_the_room_list_with_refresh_and_new_room` and `f11_us1_as4_refresh_requeries_the_rooms_without_leaving_the_list`; the refresh request is now the pure `listRoomsMessage()` in `room.js`. Also updated `f4_us1_as1_...` since the boot/retry fallback screen is now `menu: "main"` instead of `"join"`.)*

### Implementation for User Story 1

- [X] T004 [US1] Add a `#main-menu` screen to `crates/web/assets/index.html` with the three actions listed above, placed before `#room-panel`/`#directory-panel` in the navigation flow. *(the main menu also holds the logo, the "Your name" field (moved out of `#join-panel` so both Create and Join use it) and the Retry button (hidden unless the connection is lost); `#join-panel` keeps only the password and Join button.)*
- [X] T005 [US1] Add a `mainMenu` screen state and transitions to the state machine in `crates/web/assets/app.js` (`initialState`/`reduce`): boot lands on `mainMenu`; "Create new game" transitions to the existing create-room screen; "Join a game" transitions to the existing directory/room-list screen; "Options" transitions to the Options screen from User Story 3 (T009-T011) — if T009 is not yet implemented, route to the existing Sound panel as an interim target so this story remains independently shippable, and revisit once T009 lands. *(`initialState().menu` is now `"main"`; `Hello` no longer forces `menu: "directory"`; new `show-main` event. Options was routed to the Sound panel as the interim target, replaced in T028.)*
- [X] T006 [US1] Wire the directory panel's existing "New room" control (`crates/web/assets/app.js`) to return to the create-room screen rather than any now-removed direct boot path. *("New room" keeps dispatching `new-room` (create form); the create form Back button and a new directory Back button (`#directory-back`) both return to the main menu via `show-main`.)*
- [X] T007 [US1] Style `#main-menu` in `crates/web/assets/style.css` consistent with the existing panel visual language. *(`.main-menu .menu-actions` stacks the three buttons with the existing `.menu button` look.)*
- [X] T008 [US1] Run T002-T003 and confirm they pass; run the full existing `crates/web/js-tests/` suite and confirm no regression in `room.test.mjs`'s existing directory/create/join coverage. *(JS 180/0; full gate clean (fmt, clippy, `cargo test --workspace` 762/0).)*

**Checkpoint**: Main menu is live; Create/Join both reach their existing (unchanged) downstream screens.

---

## Phase 4: User Story 2 - Manage a Lobby with Bot-Filled Slots (Priority: P1)

**Goal**: Empty seats auto-fill with bots; the owner gets per-slot kick/remove-bot and a configuration panel (settings + password); everyone gets a leave icon and a two-button Ready/Start action row.

**Independent Test**: Create a 4-capacity room alone; the 3 empty seats fill with bots immediately. Remove a bot via its control; the seat shows "+"; click "+" to refill it. As owner, open the configuration icon and edit settings/password there and nowhere else. Click the leave icon to exit (spec Acceptance Scenarios 1-9).

### Tests for User Story 2

- [X] T009 [P] [US2] Add failing unit tests in `crates/core/tests/lobby.rs` for `Lobby::remove_bot_at(slot)` (clears exactly the targeted bot slot; returns a new `LobbyError::NotABot(usize)` for a non-bot slot, `LobbyError::SlotOutOfRange` out of range) and `Lobby::fill_vacant_with_bots()` (fills every `Controller::Empty` slot via `add_bot()`; no-op, no error, when none are empty). *(landed as `f11_us2_as2_remove_bot_at_clears_exactly_the_targeted_bot`, `f11_us2_remove_bot_at_rejects_a_human_an_empty_or_a_missing_slot`, `f11_us2_as4_add_bot_at_fills_exactly_the_named_empty_seat`, `f11_us2_as1_fill_vacant_with_bots_fills_every_empty_seat`, `f11_us2_fill_up_to_a_capacity_leaves_the_seats_past_it_empty`; also covers two helpers the host needs that data-model.md did not name: `add_bot_at(slot)` (the precise "+") and `fill_vacant_with_bots_up_to(seats)` (rooms with capacity < 4 must not get bots past capacity).)*
- [X] T010 [P] [US2] Add failing protocol tests in `crates/net/tests/lobby.rs` for the new `slot: u8` field on v2 `Request::AddBot`/`RemoveBot`: the host fills/clears exactly the named slot, rejects a non-owner's request, rejects an out-of-range slot, rejects `AddBot` on a non-empty slot and `RemoveBot` on a non-bot slot, each with a typed error (contracts/protocol-changes.md). *(landed in `crates/net/tests/room_host.rs` (where the raw v2 helpers live) as `f11_us2_as2_remove_bot_clears_exactly_the_named_seat`, `f11_us2_as3_only_the_owner_removes_bots_but_any_player_adds_one`, `f11_us2_bad_bot_seats_get_typed_errors`. Per FR-010 (any player may use "+") and FR-016 (which omits add-bot), `AddBot` is accepted from any member; only `RemoveBot` is owner-only. New typed codes `BadSlot`, `NotABot`, `SlotTaken`.)*
- [X] T011 [P] [US2] Add a failing test in `crates/net/tests/lobby.rs` (or `room_host.rs`) asserting a room's empty seats are bot-filled immediately after `CreateRoom`, and that a seat vacated by a human leaving pre-match is likewise auto-filled, while a seat vacated by an explicit owner `RemoveBot`/`RemovePlayer` is left `Controller::Empty` (research.md §3). *(landed in `room_host.rs` as `f11_us2_as1_a_new_room_fills_its_empty_seats_with_bots_up_to_its_capacity`, `f11_us2_edge_a_human_joining_a_bot_filled_room_takes_a_bot_seat`, `f11_us2_a_seat_left_by_a_human_before_the_match_is_refilled_with_a_bot`, `f11_us2_a_seat_emptied_by_the_owner_stays_empty_until_plus_is_used`.)*
- [X] T012 [P] [US2] Add failing tests in `crates/web/js-tests/lobby.test.mjs` for: every occupied seat shows a cross control only when the local session is the owner; every empty seat shows a "+" control for any session; the action row renders exactly "Ready" (all sessions) then "Start" (owner-only); "Add bot", "Remove bot", and "Change look" buttons are absent. *(landed as `f11_us2_as2_the_owner_sees_a_cross_on_every_other_occupied_seat`, `f11_us2_as3_a_non_owner_sees_no_cross_at_all`, `f11_us2_as4_every_empty_seat_shows_plus_for_any_session`, `f11_us2_as8_the_action_row_is_exactly_ready_then_start_for_the_owner`, `f11_us2_as9_add_bot_remove_bot_and_change_look_buttons_are_gone`. The owner gets no cross on their own seat (the host refuses to remove the owner anyway); seats past the capacity are "closed" with no "+".)*
- [X] T013 [P] [US2] Add failing tests in `crates/web/js-tests/room.test.mjs` for a configuration panel opened from a lobby-header icon, visible/usable only when `roomControls()` reports ownership, containing name/mode/size/minutes/friendly-fire/password controls and sending a single `SetRoom` request on save; and for a leave icon present in the same header area for any session. *(landed as `f11_us2_as5_as6_the_configuration_icon_is_for_the_owner_only` and `f11_us2_as5_saving_the_configuration_sends_one_set_room` (pure `roomConfigDraft`/`roomConfigReducer`/`roomConfigMessage` in `room.js`, `lobbyHeader` in `app.js`); plus `f11_us2_edge_a_bot_filled_room_stays_joinable_until_four_humans_are_in` (`roomJoinable`), because `RoomSummary.occupied` counts bots and the room list used to disable joining at `occupied >= capacity`.)*

### Implementation for User Story 2

- [X] T014 [US2] Implement `Lobby::remove_bot_at` and `Lobby::fill_vacant_with_bots` plus the new `LobbyError::NotABot(usize)` variant in `crates/core/src/lobby.rs` (depends on T009). *(also added `Lobby::add_bot_at` and `Lobby::fill_vacant_with_bots_up_to`; `fill_vacant_with_bots` returns the filled slots so the host can seat their looks.)*
- [X] T015 [US2] Add `slot: u8` to `v2::Request::AddBot`/`RemoveBot` in `crates/net/src/protocol.rs`; update the host's handler (`crates/net/src/room.rs`/`host.rs`) to validate owner identity, slot range, and slot state before calling T014's methods, returning the typed errors from T010. *(`slot` is `Option<u8>` with `serde(default)` (documented in contracts/protocol-changes.md): the web client always sends it, absent keeps the old first-free / highest-bot pick for the terminal client and the v1 `ClientMsg` translation (`client.rs`, `cli/online.rs`, `host.rs` pass `slot: None`). Validation lives in `RoomRuntime::command` (`crates/net/src/room.rs`).)*
- [X] T016 [US2] Call `Lobby::fill_vacant_with_bots` from the room host at room creation and at every pre-match slot vacancy that is not an explicit owner removal, in `crates/net/src/room.rs`/`host.rs` (depends on T011, T014). *(`RoomRuntime::fill_bots` runs in `Host::create_room` (replacing the `default_bots` loop; `--bots` no longer limits the fill). Admission takes the first empty seat within capacity, else the lowest bot seat. `remove_connection` (leave/disconnect) refills the seat; the owner kick uses `vacate_connection(_, false)`. `SetRoom`: `CapacityConflict` counts humans only, and `resize_bots` trims or adds bots. Updated about 20 pre-existing net/cli/web tests that assumed empty seats (e.g. `bots_fill_the_seats_and_humans_replace_them_up_to_the_limit`, `ready_flags_are_shared_and_leaving_gives_the_seat_to_a_bot`). Also fixed a real regression this exposed: the terminal `c` (capacity) key in `crates/cli/src/online.rs` computed the minimum from `participants()` (bots included) and could no longer shrink a room; it now counts humans.)*
- [X] T017 [US2] Rebuild the lobby header in `crates/web/assets/index.html`: add a configuration icon (owner-only) and a leave icon in the same header area; move the existing inline room-settings and password controls out of the lobby body into a new configuration panel; remove the standalone "Add bot", "Remove bot", and "Change look" buttons; reduce the action row to exactly "Ready" then "Start". *(header `.lobby-header` with `#room-config-open` (gear, owner-only, hidden otherwise) and `#leave-room` (now an icon); settings and password moved to a new `#room-config` dialog; the action row is `.lobby-actions` with `#ready` then `#start`. `#lobby-password-set` was replaced by the panel's single Save and `#lobby-password-remove` became a checkbox.)*
- [X] T018 [US2] Implement the configuration panel's open/save/cancel logic in `crates/web/assets/room.js`/`app.js`, reusing the existing `roomControls()` ownership gate, sending one `SetRoom` request on save (contracts/protocol-changes.md) and discarding local edits on cancel. *(the panel is a navigation layer with a local draft; Save sends `roomConfigMessage(draft)` (one `SetRoom`), Cancel/Escape drops it; the panel closes itself if ownership moves while it is open (spec Edge Case).)*
- [X] T019 [US2] Add per-slot cross/plus controls to the lobby seat rendering in `crates/web/assets/app.js`: cross sends `RemovePlayer{membership_id}` for a human seat or `RemoveBot{slot}` for a bot seat, visible only when the local session owns the room; "+" sends `AddBot{slot}`, visible on any empty seat for any session. *(cross/plus are `.seat-control` buttons built from `lobbyCards(lobby, you, { capacity, memberships })`; `RoomUpdated.memberships` is now kept in state (`state.memberships`) to name the human to kick.)*
- [X] T020 [US2] Wire the new leave icon to the existing `LeaveRoom` flow in `crates/web/assets/app.js`. *(the existing `#leave-room` id was kept, so the existing confirm-dialog wiring is reused unchanged.)*
- [X] T021 [US2] Style the lobby header icons, per-slot controls, configuration panel, and two-button action row in `crates/web/assets/style.css`.
- [X] T022 [US2] Run T009-T013 and confirm they pass; run `crates/core/tests/lobby.rs`, `crates/net/tests/{lobby,room_host}.rs`, and `crates/web/js-tests/{lobby,room}.test.mjs` in full and confirm no regression. *(full gate clean: fmt, clippy, `cargo test --workspace` 774/0, JS 193/0 (run together with the US3 gate, since US3 red tests were already written when US2 was verified).)*

**Checkpoint**: Lobby seats auto-fill, per-slot controls work and are owner-gated server-side, configuration lives in one panel, and the action row is exactly Ready/Start.

---

## Phase 5: User Story 3 - Configure Settings from Tabbed Options (Priority: P2)

**Goal**: One Options screen with Customize/Sound/Controls tabs, reachable from the main menu and from the in-game pause menu.

**Independent Test**: Open Options from the main menu and from the pause menu; both land on the same tabbed screen; switching tabs preserves unsaved edits made in another tab (spec Acceptance Scenarios 1-5).

### Tests for User Story 3

- [X] T023 [P] [US3] Add failing tests in `crates/web/js-tests/app.test.mjs` asserting the Options screen renders exactly three tabs in order (Customize, Sound, Controls); the Sound tab shows the existing sound controls; the Controls tab shows the existing control bindings; switching away from and back to a tab preserves in-progress (unsaved) edits made on it during the session. *(landed as `f11_us3_as1_options_has_exactly_three_tabs_in_order`, `f11_us3_as2_as3_the_sound_and_controls_tabs_hold_the_existing_controls`, `f11_us3_switching_tabs_only_changes_which_panel_shows` (tab state is the pure `optionsReducer`/`optionsView`; panels are only hidden, never rebuilt, so edits survive).)*
- [X] T024 [P] [US3] Add a failing test in `crates/web/js-tests/app.test.mjs` asserting the in-game pause menu's Sound entry opens Options on the Sound tab and its Controls entry opens Options on the Controls tab. *(landed as `f11_us3_pause_menu_sound_and_controls_open_the_matching_tab` (`HUB_OPTIONS_TABS`), plus `f11_us1_as5_us3_as4_the_main_menu_opens_options_on_customize`.)*

### Implementation for User Story 3

- [X] T025 [US3] Build a tabbed Options container in `crates/web/assets/index.html` wrapping the existing sound-panel and controls-panel markup as two tabs, plus a placeholder/empty Customize tab for User Story 4 to fill in (T031). *(`#options-panel` (dialog) with a `role="tablist"` and three `role="tabpanel"` sections: `#options-customize` (placeholder until T039), `#sound-panel`, `#controls-panel`. The per-panel Close buttons (`#close-sound`, `#controls-close`) became one `#options-close`. Updated `crates/web/tests/http.rs` (`the_navigation_hub_and_settings_panels_are_labelled_dialogs`) for the dialog/tabpanel split.)*
- [X] T026 [US3] Implement tab-switching state in `crates/web/assets/app.js` that keeps each tab's in-progress edits across switches within the same Options session. *(`openOptions(tab)`/`selectOptionsTab`; Arrow Left/Right move between tabs; leaving the Controls tab cancels a pending key capture.)*
- [X] T027 [US3] Update the pause menu's Sound and Controls entries in `crates/web/assets/app.js` to open the Options screen defaulting to the matching tab, replacing today's direct-panel-open behavior. *(the header Sound/Controls buttons also open Options on their tab; Options is a navigation layer above the pause hub, so closing it returns to the paused game (AS-5).)*
- [X] T028 [US3] Also route the Main Menu's "Options" action (T005) to this tabbed screen if T005 used an interim target. *(Main Menu Options now opens the tabbed screen on Customize; the interim Sound-panel wiring from T005 was removed.)*
- [X] T029 [US3] Style the tab bar in `crates/web/assets/style.css`.
- [X] T030 [US3] Run T023-T024 and confirm they pass; run the full `crates/web/js-tests/` suite and confirm no regression in existing sound/controls coverage. *(JS 193/0; full gate clean (see T022).)*

**Checkpoint**: Options is one tabbed screen reachable from both entry points; Customize tab is an empty placeholder awaiting User Story 4.

---

## Phase 6: User Story 4 - Customize Without the Gender Choice (Priority: P2)

**Depends on**: User Story 3's Options/Customize tab container (T025) existing to host this panel.

**Goal**: Customize tab offers Head/Face/Costume with no gender selector, and a small randomize icon on the character preview instead of a standalone "Random" button.

**Independent Test**: Open Customize; confirm no gender control; confirm Head/Face/Costume selectors; confirm the preview's corner icon randomizes colour/head/face/costume together (spec Acceptance Scenarios 1-5).

### Tests for User Story 4

- [X] T031 [P] [US4] Add failing tests in `crates/net/tests/look.rs` asserting `eksplode_re_net::look::Look` has no settable `gender` field, has a new `costume: Costume` field with `Costume::ALL` and a sensible default, and that `Look`'s lenient JSON decode silently ignores a legacy `gender` key in an incoming payload rather than erroring (mirrors the existing "a bad/old entry only ever gives a default" doctrine). *(rewrote `crates/net/tests/look.rs` around the new shape (defaults, option lists incl. `Costume::ALL` = None/Scarf/Cape/BowTie/Medal and `Costume::default()`, round trip, lenient decode) and added `f11_us4_as5_a_legacy_look_with_a_gender_still_decodes_with_a_default_costume` (both `from_wire` and the typed serde reader ignore `gender`; nothing sends it). Failed first on the missing `Costume`.)*
- [X] T032 [P] [US4] Add a failing test in `crates/net/tests/looks.rs` asserting a `SetLook` request carrying `{"gender": "Female", ...}` from a legacy client is accepted (the key is discarded) and the resulting broadcast `Look` carries the sender's `costume` choice. *(landed as `f11_us4_a_legacy_set_look_with_a_gender_is_accepted_and_the_costume_is_broadcast`; also moved `a_chosen_look_is_in_the_welcome_and_in_every_lobby_broadcast` to a costume.)*
- [X] T033 [P] [US4] Add a failing test in `crates/web-render/tests/look.rs` (or `iso3d.rs`) asserting `web_render::look::Look`'s existing `gender`/`Gender` field and its slot-parity default in `default_for` are unchanged, and that a new `costume` field defaults per-slot via `from_value`'s existing missing-field fallback when absent from the payload — i.e., no gender regression, costume follows the same lenient pattern Head/Face already use. *(landed in `crates/web-render/tests/look.rs` as `f11_us4_the_costume_names_match_the_server`, `f11_us4_as5_a_missing_costume_or_gender_falls_back_to_the_slot_default` (slot-parity gender unchanged), `f11_us4_every_costume_changes_the_picture_and_has_its_colours` (every view and frame), plus `f11_us4_every_costume_changes_the_3d_character` in `iso3d.rs`; the `Look { .. }` literals in look.rs, sprites3.rs and iso3d.rs gained `costume: Costume::None`.)*
- [X] T034 [P] [US4] Add failing tests in `crates/web/js-tests/look.test.mjs` asserting: the editor has no gender/"Look" option; `COSTUMES`/costume selection exists; `editorReducer`'s randomize action sets colour, head, face, and costume (not gender, which is no longer part of the editable/transmitted draft). *(rewrote the gender fixtures in `look.test.mjs` to costumes and added `f11_us4_as5_a_look_saved_with_a_gender_loads_without_error_and_a_default_costume`, `f11_us4_as3_randomize_sets_colour_head_face_and_costume`, `f11_us4_as1_as2_as4_the_customize_tab_has_head_face_costume_and_a_corner_randomize_icon`.)*

### Implementation for User Story 4

- [X] T035 [US4] In `crates/net/src/look.rs`: add `Costume` via the existing `options!` macro with an initial option list (content choice, per research.md §5); add `costume: Costume` to `Look`; remove `gender`/`Gender` from `Look`'s public/settable surface; update `Look::default_for` and the lenient JSON reader accordingly (depends on T031). *(content choice: `Costume { None, Scarf, Cape, BowTie, Medal }`. `Gender` was removed from `net::look` entirely (nothing on the server reads it any more). `costume` has `#[serde(default)]` so looks from older hosts still decode; `Default for Costume` is a manual impl with a scoped `clippy::derivable_impls` allow, because the `options!` macro cannot take `#[default]`.)*
- [X] T036 [US4] In `crates/web-render/src/look.rs`: add `Costume` and the `costume` field to `web_render::look::Look` with a `from_value` fallback to `default_for(slot).costume` when absent, following the existing `head`/`face` pattern; leave the existing `gender`/`Gender` field and `default_for`'s slot-parity default untouched (depends on T033). *(`Costume`, `COSTUME_NAMES` and `costume` added; `gender` and `default_for` slot parity untouched, so a page that no longer sends `gender` gets the alternating silhouette from `from_value`'s existing fallback.)*
- [X] T037 [US4] In `crates/web-render/src/iso3d.rs`: render a costume overlay alongside the existing head/face overlays, using new sprite colour constants analogous to `CAP`/`HAT`/`FRAME` etc. *(the 2D grids (`costume_into` in `look.rs`: `r` scarf, `v` cape, `w` bow tie, `i` ribbon, `d` medal, with view-aware placement; bow tie and medal hidden from behind, cape edges only from the front) and the 3D character (`iso3d.rs`, cape cube behind the body, the others in front) use new colours `SCARF`, `CAPE`, `BOWTIE`, `RIBBON`, `MEDAL`. WASM rebuilt with `scripts/build-wasm.sh` (219914 bytes).)*
- [X] T038 [US4] In `crates/web/assets/look.js`: remove `gender` from `OPTIONS`, `editorReducer`, and `defaultLook()`'s transmitted draft; add `COSTUMES`/costume handling in the same places; update the randomize action to set colour/head/face/costume. *(`GENDERS` export removed; `sanitizeLook` drops a saved `gender`; random draws colour, head, face and costume.)*
- [X] T039 [US4] In `crates/web/assets/index.html`: fill the Customize tab placeholder from T025 — remove the `#gender` select, add a `#costume` select next to `#head`/`#face`, remove the standalone "Random" button, and add a small randomize icon anchored to the bottom-right corner of the character preview. *(the standalone `#editor` dialog and its `#done-look` button are gone (Options Close replaces Done); `#random-look` is now a `.randomize-icon` inside `.preview-wrap` with the preview canvas.)*
- [X] T040 [US4] In `crates/web/assets/app.js`: replace the `el.gender` wiring (`fillSelect`, value sync, change listener) with equivalent `el.costume` wiring; wire the new randomize icon to the existing randomize action. *(also removed the old "editing" state (it belonged to the removed Change look button); the preview only paints while the Customize tab is visible. `crates/web/tests/consistency.rs` now checks `COSTUMES` across net, web-render, look.js and PROTOCOL.md (whose Looks table was updated here because the test needs it), and `crates/web/tests/looks.rs` was moved to costume.)*
- [X] T041 [US4] Style the randomize icon overlay in `crates/web/assets/style.css`.
- [X] T042 [US4] Run T031-T034 and confirm they pass; run `crates/net/tests/{look,looks}.rs`, `crates/web-render/tests/{look,iso3d}.rs`, and `crates/web/js-tests/look.test.mjs` in full and confirm no regression beyond the intended gender-selector removal. *(full gate clean: fmt, clippy, `cargo test --workspace` 780/0, JS 196/0.)*

**Checkpoint**: Customize tab has no gender control, offers Head/Face/Costume, and randomizes via the preview's corner icon; other players still render with slot-consistent silhouette variety.

---

## Phase 7: User Story 5 - See a Ranked Scoreboard (Priority: P2)

**Goal**: End-of-match scoreboard ranks players by survival order (last-alive first, then reverse elimination order) and shows only name/kills/bombs placed.

**Independent Test**: Finish a 4-player match with a known elimination order; the scoreboard lists the last-alive player first, others in reverse elimination order, each row showing only name/kills/bombs placed (spec Acceptance Scenarios 1-4).

### Tests for User Story 5

- [X] T043 [P] [US5] Add a failing unit test in `crates/core/tests/bombs.rs` asserting a player's `bombs_placed` increments exactly once per successful placement (`crates/core/src/bomb.rs`), independent of whether the bomb later detonates, is defused, or is kicked. *(landed as `f11_us5_bombs_placed_counts_each_successful_placement_once` (a refused placement on an occupied tile, the later explosion and a test-only `add_bomb` do not count). Detonation is covered here; kicked/defused variants go through the same single increment site, so they are not tested separately.)*
- [X] T044 [P] [US5] Add a failing unit test in `crates/core/tests/chain.rs` (or a new scenario in `round_end.rs`) asserting: when player A's bomb eliminates player B, `B.eliminated_order == Some(1)` (next unused ordinal) and `A.kills == 1`; when a player's own bomb eliminates only themselves, nobody's `kills` increments; when two players share a tile and both are caught in the same blast, both get a stamped `eliminated_order` and the contributing non-victim player(s) get credited per `Flame.owners` (data-model.md "Match Statistics"). *(landed in `chain.rs` as `f11_us5_as1_a_kill_is_credited_to_the_bomb_owner_and_stamps_the_order`, `f11_us5_a_self_elimination_credits_nobody`, `f11_us5_as4_players_caught_in_one_tick_share_their_order_and_later_ones_come_after`, `f11_us5_every_bomb_whose_flame_reaches_the_victim_is_credited`. Design decision: players eliminated in the same tick share one ordinal (the next unused one); the scoreboard tie-break then orders them.)*
- [X] T045 [P] [US5] Add a failing unit test in `crates/core/tests/closing.rs` asserting the closing-wall elimination site (`crates/core/src/closing.rs:125`) also stamps `eliminated_order` consistently with the blast site. *(landed as `f11_us5_a_crushed_player_gets_the_next_elimination_order_and_nobody_a_kill` (a landing block is nobody's kill).)*
- [X] T046 [P] [US5] Add failing protocol tests in `crates/net/tests/consistency.rs` (or `room.rs`) round-tripping the new additive `kills`/`bombs_placed`/`eliminated_order` fields on the existing per-player snapshot/result message, and asserting a payload missing these fields (simulating an old peer) decodes with safe defaults (0/0/null) rather than erroring. *(landed in `crates/net/tests/codec.rs` (which has the round-trip helper; `consistency.rs` is a latency/agreement integration test) as `f11_us5_match_statistics_ride_on_the_snapshot_and_default_for_old_hosts`.)*
- [X] T047 [P] [US5] Add failing tests in `crates/web/js-tests/app.test.mjs` for `resultRows()`/the results table: header is exactly "Player", "Kills", "Bombs placed"; row order is last-alive/winner first, then reverse `eliminated_order`, then a deterministic tie-break (kills descending, then original slot ascending) for equal/absent cases (data-model.md). *(landed as `f11_us5_as2_the_scoreboard_has_exactly_player_kills_and_bombs_placed`, `f11_us5_as1_the_last_alive_comes_first_then_the_reverse_elimination_order`, `f11_us5_as3_as4_ties_break_by_kills_then_by_slot`; the pre-existing `us3_as1_...`/`us3_as2_...` result tests were rewritten for the new columns (the latter is now `us3_as2_sc004_the_winner_is_first_for_draw_team_and_chaser_outcomes`, covering SC-004 across Normal/Team/Chaser).)*

### Implementation for User Story 5

- [X] T048 [US5] Add `kills: u32`, `bombs_placed: u32`, `eliminated_order: Option<u32>` to `core::state::Player` in `crates/core/src/state.rs`, reset to `0`/`0`/`None` when a match starts from the lobby. *(all three fields have `serde(default)`; `GameState::custom`/`new` start them at 0/0/None, and every match builds a fresh `GameState`, so they reset per match. Regenerated `crates/web-render/tests/fixtures/observation.json` with `UPDATE_FIXTURES=1` (the diff is only the three new fields per player).)*
- [X] T049 [US5] Increment the placing player's `bombs_placed` at the bomb-placement site in `crates/core/src/bomb.rs` (depends on T043).
- [X] T050 [US5] At the blast elimination site `crates/core/src/step.rs:185`, stamp the victim's `eliminated_order` to the next unused ordinal and increment `kills` for every other player whose bomb contributed via that tile's `Flame.owners` bitmask, with no increment for a self-elimination (depends on T044). *(kills come from the OR of the `Flame.owners` of every hurting flame on the victim's tile (the victim's own bit is skipped). The ordinal is stamped once per tick by a new `stamp_eliminations` at the end of the elimination phase of `step`, comparing alive flags from the start of the tick.)*
- [X] T051 [US5] At the closing-wall elimination site `crates/core/src/closing.rs:125`, stamp `eliminated_order` consistently with T050 (depends on T045). *(no edit in `closing.rs` was needed: `closing::land` runs inside `step` (via `closing::advance`), so the same `stamp_eliminations` pass stamps crushed players consistently with blast victims.)*
- [X] T052 [US5] Relay `kills`/`bombs_placed`/`eliminated_order` through the existing per-player snapshot/result message in `crates/net/src/protocol.rs` as additive fields (depends on T046). *(no protocol change was needed: `Participant` is already relayed inside `Observation` in `ServerMsg::Snapshot`/`Reply::Snapshot`, so the fields travel as additive JSON with `serde(default)` for old peers.)*
- [X] T053 [US5] Implement the new sort key and row shape in `resultRows()` in `crates/web/assets/app.js`: last-alive/winner first (no `eliminated_order`), then descending `eliminated_order`, then the documented tie-break; drop the Result/Bombs/Range/Speed/Powers columns in favor of Player/Kills/Bombs placed. *(the winning side comes first (Winner; the winning team including fallen members; the chaser on ChaserWins; surviving runners on RunnersWin), then players still alive, then descending `eliminated_order`, then kills descending, then slot ascending. A missing or invalid field from an old host reads as 0/null. Row shape `{slot, label, kills, bombsPlaced}`.)*
- [X] T054 [US5] Update the `#results-panel` table header in `crates/web/assets/index.html` to "Player", "Kills", "Bombs placed".
- [X] T055 [US5] Run T043-T047 and confirm they pass; run `crates/core/tests/{bombs,chain,closing,round_end}.rs`, `crates/net/tests/consistency.rs`, and `crates/web/js-tests/app.test.mjs` in full and confirm no regression. *(all US5 tests pass; `cargo test --workspace --no-fail-fast` shows no failures beyond the fixture regenerated in T048; fmt/clippy clean; JS 199/0. The `pty.rs` check failed in two runs while `crates/cli/src/args.rs` was being changed concurrently by an unrelated fix, and passes now.)*

**Checkpoint**: Scoreboard ranks by survival order and shows only name/kills/bombs placed, backed by deterministic `core` bookkeeping.

---

## Phase 8: User Story 6 - No Red Focus Outline (Priority: P3)

**Goal**: Focused interactive elements no longer show a red outline; a non-red focus indicator remains.

**Independent Test**: Tab through the main menu, lobby, and options screens; no red outline appears; some visible focus indicator remains (spec Acceptance Scenarios 1-2).

### Tests for User Story 6

- [X] T056 [P] [US6] Add a failing assertion in `crates/web/js-tests/nav-focus.test.mjs` (or a new focused CSS check) that no `:focus-visible` rule in `crates/web/assets/style.css` sets `outline-color`/`outline` to `#d43b32` (or an equivalent red), while at least one `:focus-visible` rule still sets a visible, non-transparent outline. *(landed as `f11_us6_as1_as2_focus_visible_outlines_are_visible_and_not_red` in `nav-focus.test.mjs`: parses every `:focus-visible` rule, resolves `var(--x)` against `:root`, rejects red values and requires a visible solid outline on buttons. Failed first on `#d43b32`.)*

### Implementation for User Story 6

- [X] T057 [US6] In `crates/web/assets/style.css`, change the `:focus-visible` outline color away from `#d43b32` to a non-red color already used in the stylesheet's palette (e.g. an existing accent/ink custom property), keeping the existing outline width/offset so visibility is preserved. *(`outline: 3px solid var(--slot0)` (the palette's accent blue `#4d63d8`), with width, offset and the white `box-shadow` ring unchanged.)*
- [X] T058 [US6] Run T056 and confirm it passes; run the full `crates/web/js-tests/` suite and confirm no regression in existing `nav-focus.test.mjs` coverage. *(full gate clean: fmt, clippy, `cargo test --workspace --no-fail-fast` 789/0, JS 200/0.)*

**Checkpoint**: No red focus outline anywhere; focus remains visible.

---

## Phase 9: Polish & Cross-Cutting Concerns

- [X] T059 [P] Update `README.md`/`WEB.md` (or equivalent docs under `game/`) for the new Main Menu → Lobby → Options navigation, the per-slot bot-fill/kick/add behavior, the configuration panel, the Customize tab's Head/Face/Costume options, and the new scoreboard columns/order. *(updated README.md (features: main menu, Options tabs, Customize head/face/costume plus the randomize icon, bot-filled seats with cross, "+" and gear; the scoreboard paragraph), docs/WEB.md (main menu, Lobby, Options, Customize, scoreboard, focus colour; `--bots` prose note, while the options table stays generated from the usage text as `docs.rs` requires), docs/ARCHITECTURE.md (ownership: AddBot open to all, bot fill, match statistics; look modules), docs/GAMEPLAY.md (Team bots, costume) and docs/DEVELOPMENT.md (look categories, how to add a per-player statistic, the `UPDATE_FIXTURES=1` refresh).)*
- [X] T060 [P] Document the `AddBot`/`RemoveBot` `slot` parameter addition in the protocol documentation referenced by `contracts/protocol-changes.md` (e.g. `specs/008-multi-room-servers/contracts/room-protocol.md` or wherever the canonical v2 protocol reference lives). *(canonical reference is docs/PROTOCOL.md: new v2 subsections "Lobby seats and bots" (`AddBot`/`RemoveBot {slot?}`, who may send them, `BadSlot`/`NotABot`/`SlotTaken`/`Full`, bot-fill and capacity rules) and "Match statistics" (`kills`, `bombs_placed`, `eliminated_order`); the Looks table gained `costume` and lost `gender` (in T040). The legacy v1 `AddBot`/`RemoveBot` rows are unchanged, per contracts/protocol-changes.md.)*
- [ ] T061 Run the full quickstart.md browser-smoke matrix (all 8 scenarios) end-to-end against a locally running server and record results. *(NOT done: no browser was available during implementation. The DOM/CSS was reviewed against contracts/ui-screens.md and every screen is covered by structural JS tests, but all 8 quickstart scenarios still need a real browser run by a human before merge, especially the seat cross/"+" placement, the Options dialog layout and the randomize icon overlay.)*
- [X] T062 Run the full merge gate: `cargo fmt --check`, `cargo clippy --all-targets -- -D warnings`, `cargo test --workspace`, and the complete `crates/web/js-tests/` suite; compare against the T001 baseline and confirm zero regressions. *(final gate: `cargo fmt --check` clean, `cargo clippy --all-targets -- -D warnings` clean, `cargo test --workspace --no-fail-fast` 790/0 (run twice), `node --test` 200/0; baseline was 762/0 and 175/0. Two fixes came out of this pass. (1) `crates/cli/tests/pty.rs` failed intermittently at "joined non-owner sees Only Host": `scripts/pty-enter-check.py` pressed Enter as soon as the "Rooms" header showed "No rooms available" (a pre-existing race), so it now waits for the listed room ("Room:"); 20/20 standalone after the fix. (2) The terminal room list showed bot-filled rooms as "4/4"; it now shows humans and bots, "1/4 +3 bots" (`crates/cli/src/online.rs`, test `f11_the_room_list_counts_humans_and_shows_the_bots_apart`).)*

### Post-delivery review (orchestrating session, after T062)

An 8-angle automated code review of the full diff plus independent manual spot-checks found one real
correctness bug (three review angles converged on it independently, one with a live reproduction) and
several non-blocking reuse/simplification/efficiency suggestions (not fixed; left for a future cleanup
pass since none are correctness issues): duplicated "humans in room" computation across `room.js`/`app.js`,
positional (not `id`-keyed) access into `lobbyActions()`, a redundant bot-removal loop in `resize_bots`,
and `roomControls(state)`/lobby-card recomputation inside the per-tick `updateUi()` path.

- [X] **Fixed**: `SetRoom`'s capacity-shrink check (`crates/net/src/room.rs`, the `Request::SetRoom` arm)
  counted humans in aggregate (`self.humans() `) rather than by position, so an owner-transfer-then-kick
  sequence could strand a connected human at a slot index at or beyond a newly shrunk capacity —
  invisible to `admission_slot`'s capacity window — letting the room silently admit more humans than its
  declared capacity (reproduced and confirmed independently by three review passes). Replaced with a
  check that rejects the shrink whenever any human occupies a slot at or beyond the new capacity,
  positioned after the existing `Self::new(..)` dry-run (which already bounds `capacity` to `1..=4`, so
  the new slice-based check can't panic on an out-of-range value). Removed the now-unused private
  `humans()` helper. Added a failing-then-passing regression test,
  `set_room_capacity_cannot_strand_a_human_beyond_the_new_window`, in `crates/net/tests/room_host.rs`,
  reproducing the exact scenario (owner kicks two mid-slot humans, leaving a human stranded at slot 3,
  then shrinks capacity to 2). Documented in `contracts/protocol-changes.md`.
- [X] Fixing the above also surfaced that 7 existing `crates/net/tests/room_host.rs` tests were stale
  against FR-008 (every room now auto-fills to capacity with bots immediately on creation, unconditionally
  — not just when explicitly configured): `v2_players_choose_independent_rooms_without_implicit_admission`,
  `v2_invalid_admissions_and_room_limits_do_not_mutate_existing_rooms`, and
  `protected_room_admission_verifies_passwords_without_public_secrets` asserted stale `occupied` counts
  (now `capacity`, not `1`); `in_process_host_controls_require_the_selected_room_reference` called the
  in-process `add_room_bot` API on an already-full room (now frees a seat first);
  `f4_us1_as5_a_start_with_too_few_participants_is_refused_with_the_missing_count` and
  `a_chaser_room_starts_with_four_participants_and_one_chaser` assumed a room starts empty and bots are
  added one at a time (now strips bots down from the auto-filled room instead, to exercise the same
  "N missing" messaging); `v2_start_addbot_and_removebot_require_the_room_owner` asserted `AddBot` requires
  ownership, which FR-010 explicitly changed (now asserts a non-owner's `AddBot` is refused as `Full`
  against the already-full room, then proves it succeeds once the owner frees a seat). All 46
  `room_host.rs` tests pass, confirmed stable across repeated runs.

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: no dependencies.
- **Foundational (Phase 2)**: none — no story blocks another (see Phase 2 note).
- **User Stories (Phases 3-8)**: each can start once Phase 1 is done. **Exception**: User Story 4 (Phase 6) needs User Story 3's Options/Customize tab container (T025) to exist before T039 can place the Customize markup — if US4 is implemented before US3, its tests (T031-T034) can still be written and its `net`/`web-render` work (T035-T038) completed, but T039-T041 (wiring the panel into the Options screen) must wait for T025.
- **Polish (Phase 9)**: after all desired stories are complete.

### Within Each User Story

- Tests (first subsection) MUST be written and observed failing before the implementation subsection.
- `core`/`net` changes precede the `web` JS/HTML/CSS changes that depend on them (e.g., T014-T016 before T017-T021 in US2; T035-T037 before T038-T041 in US4; T048-T052 before T053-T054 in US5).
- Each story's final task re-runs its own tests plus any existing suite it touches, to catch regressions before moving on.

### Parallel Opportunities

- All `[P]`-marked test tasks within a phase target different files and can be written together.
- US1, US2, US3, US5, and US6 can be implemented in parallel by different contributors once Phase 1 is done, since they touch disjoint `core`/`net` surfaces and largely disjoint regions of `app.js`/`index.html`/`style.css`. US4 can start in parallel too, but its final wiring step (T039-T041) should land after US3's T025.
- T059-T060 (docs) can run in parallel with each other and with T061-T062.

---

## Parallel Example: User Story 2

```bash
# Tests, in parallel (different files):
Task: "T009 Lobby::remove_bot_at/fill_vacant_with_bots unit tests in crates/core/tests/lobby.rs"
Task: "T010 AddBot/RemoveBot slot protocol tests in crates/net/tests/lobby.rs"
Task: "T012 per-slot control tests in crates/web/js-tests/lobby.test.mjs"
Task: "T013 configuration-panel/leave-icon tests in crates/web/js-tests/room.test.mjs"
```

---

## Implementation Strategy

### MVP First (User Stories 1 and 2)

1. Complete Phase 1 (Setup).
2. Complete Phase 3 (US1: Main Menu) — gives every other story a coherent entry point, though not strictly required by them.
3. Complete Phase 4 (US2: Lobby) — the highest-value, most-requested redesign.
4. **STOP and VALIDATE**: run both stories' independent tests and the relevant quickstart.md scenarios.

### Incremental Delivery

1. Setup → US1 (menu) → demo.
2. Add US2 (lobby) → demo.
3. Add US3 (tabbed options) → demo.
4. Add US4 (customize) → demo (requires US3's container).
5. Add US5 (scoreboard) → demo.
6. Add US6 (focus outline) → demo.
7. Polish (Phase 9) → merge.

### Parallel Team Strategy

After Phase 1: one contributor on US2 (lobby, the largest story), one on US1+US3 (navigation shell, since US3 unblocks US4's final wiring), one on US5 (scoreboard, fully disjoint `core` surface), one on US4 (net/web-render work can start immediately; final wiring waits on US3) and US6 (trivial, fits in any spare cycle).
