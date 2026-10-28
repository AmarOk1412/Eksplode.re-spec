---
description: "Implementation tasks for mobile browser gameplay"
---

# Tasks: Mobile Browser Gameplay

**Input**: Design documents from `/specs/007-mobile-gameplay/`

**Prerequisites**: plan.md, spec.md, research.md, data-model.md, contracts/touch-controls.md, quickstart.md

**Tests**: Included before implementation as required by the project constitution and plan. Each acceptance scenario is mapped in `quickstart.md`.

**Organization**: Tasks are grouped by user story so each increment can be implemented and validated independently.

## Phase 1: Setup

**Purpose**: Establish the shared embedded asset route required by the mobile input module.

- [X] T001 Add a failing HTTP test for the `/touch.js` JavaScript route and content type in `game/crates/web/tests/http.rs`.
- [X] T002 Create the `touch.js` module shell in `game/crates/web/assets/touch.js` and register its embedded route in `game/crates/web/src/assets.rs`; make T001 pass.

## Phase 2: Foundational

**Purpose**: No additional shared foundation is needed. T001-T002 establish the testable embedded-module path; story-specific behavior follows below.

## Phase 3: User Story 1 - Join and manage a game on a phone (Priority: P1)

**Goal**: Use a phone browser to complete room setup and lobby actions, with layouts that remain usable across phone viewport changes and without regressing desktop controls.

**Independent Test**: Using a touch-capable mobile viewport, complete room creation or joining, player entry and lobby setup; verify controls remain in bounds, then verify the existing desktop keyboard, mouse and controller flow.

### Tests for User Story 1

- [X] T003 [P] [US1] Extend `game/scripts/browser-smoke.mjs` to cover mobile-viewport room setup and lobby touch interactions, portrait/landscape viewport changes, in-bounds controls, and the desktop keyboard Start regression path.

### Implementation for User Story 1

- [X] T004 [P] [US1] Document mobile-browser access, phone setup and lobby use, portrait/landscape layouts, touch controls and compatibility limits in `game/README.md` and `game/docs/WEB.md`.
- [X] T005 [US1] Adapt room creation, joining and lobby layouts for phone viewport sizes and the on-screen text-entry keyboard in `game/crates/web/assets/style.css`.

**Checkpoint**: A phone player can reach a match through the existing touch-operable setup and lobby; desktop input remains unchanged.

## Phase 4: User Story 2 - Play a match with touchscreen controls (Priority: P1)

**Goal**: Play in portrait using a bottom-left joystick and bottom-right Bomb/Punch actions; retain the existing landscape layout and show Punch only when powered.

**Independent Test**: Play a phone match in portrait with controls at both bottom corners, then rotate to landscape and verify the existing layout. Confirm Punch is absent until the player has the power and all held actions clear on orientation changes.

### Tests for User Story 2

- [X] T006 [P] [US2] Add Node tests for joystick direction/neutral mapping, pointer ownership, one-shot Bomb/Punch presses, simultaneous pointers and release/cancellation in `game/crates/web/js-tests/touch.test.mjs`.
- [X] T007 [P] [US2] Add HTTP markup assertions for mobile control zones, conditional Punch and responsive match markup in `game/crates/web/tests/http.rs`.
- [X] T008 [P] [US2] Extend `game/scripts/browser-smoke.mjs` with portrait touch input, bottom-corner placement, landscape layout regression, multi-touch actions and pointer-cancel/focus-loss checks.

### Implementation for User Story 2

- [X] T009 [US2] Implement the pure pointer/joystick tracker and one-shot action state in `game/crates/web/assets/touch.js` to satisfy T006, preserving one-cardinal-direction input and pointer ownership.
- [X] T010 [P] [US2] Add match-only touch controls with conditional Punch to `game/crates/web/assets/index.html`; style portrait bottom-corner placement and preserve landscape placement in `game/crates/web/assets/style.css` to satisfy T007.
- [X] T011 [US2] Integrate the touch tracker and touch-capable layout in `game/crates/web/assets/app.js`; merge touch movement/actions into the existing input tick and clear held gameplay input on orientation change, touch cancellation, blur, hidden page or open navigation layer.

**Checkpoint**: A phone player can complete a match using only touch while desktop controls, game rules and multiplayer behavior remain unchanged.

## Phase 5: User Story 2 Follow-Up - Browser Compatibility and Orientation (Priority: P1)

**Goal**: Enable touch-capable browsers such as Firefox for Android and keep input/layout responsive across portrait/landscape orientation changes.

**Independent Test**: Simulate Firefox Android touch-event capability without coarse-pointer media data, and verify controls appear in portrait and held input clears as the layout changes to landscape.

### Tests for User Story 2 Follow-Up

- [X] T012 [P] [US2] Add Node tests for touch-capability detection from coarse pointers, `maxTouchPoints` and touch events in `game/crates/web/js-tests/touch.test.mjs`.
- [X] T013 [P] [US2] Extend `game/scripts/browser-smoke.mjs` to emulate Firefox Android touch-event capability and verify portrait and landscape layouts remain interactive.

### Implementation for User Story 2 Follow-Up

- [X] T014 [US2] Implement capability-based touch detection from pointer/touch-event signals, keep gameplay enabled in portrait, and clear stale input on orientation changes in `game/crates/web/assets/touch.js` and `game/crates/web/assets/app.js`.

**Checkpoint**: Firefox Android/hybrid touch is recognized without user-agent checks; portrait and landscape layouts remain interactive.

## Phase 6: User Story 3 - Review the Round Result and Return to Lobby (Priority: P1)

**Goal**: Keep the outcome and final standings/stats visible after the round and provide an explicit Return to Lobby action.

**Independent Test**: Deliver an `Ended` outcome followed by the lobby message, verify the result and all occupied scoreboard rows remain visible, then activate Return to Lobby and verify the latest lobby appears.

### Tests for User Story 3

- [X] T015 [P] [US3] Add app tests for winner/draw/team/chaser result rows, retention through a lobby message, and dismissal only through Return to Lobby in `game/crates/web/js-tests/app.test.mjs`.
- [X] T016 [P] [US3] Add HTTP markup assertions for the result panel, final scoreboard and accessible Return to Lobby button in `game/crates/web/tests/http.rs`.
- [X] T017 [P] [US3] Extend `game/scripts/browser-smoke.mjs` to verify result visibility after the lobby update and explicit return to the lobby.

### Implementation for User Story 3

- [X] T018 [US3] Retain the final outcome/observation in app state, keep stage `results` through the lobby broadcast, and derive standings/stat rows from the final observation in `game/crates/web/assets/app.js`.
- [X] T019 [P] [US3] Add the accessible result panel, scoreboard table and Return to Lobby control in `game/crates/web/assets/index.html`; style persistent result presentation in `game/crates/web/assets/style.css`.
- [X] T020 [US3] Render the final outcome and scoreboard, wire Return to Lobby to reveal the latest lobby and restore focus in `game/crates/web/assets/app.js`.
- [X] T021 [P] [US3] Document Firefox Android support, portrait/landscape controls, conditional Punch and the post-round scoreboard/return flow in `game/README.md` and `game/docs/WEB.md`.

**Checkpoint**: All participants can review the round outcome and standings until they choose to return to the lobby.

## Phase 7: Polish and Cross-Cutting Concerns

**Purpose**: Rerun the documented checks and complete manual device verification.

- [X] T022 Run the automated and browser commands in `specs/007-mobile-gameplay/quickstart.md` from `game/`; record final outcomes in that guide.
- [X] T023 Manually verify Firefox Android, portrait bottom-corner controls, landscape layout, conditional Punch, results persistence, safe areas, multi-touch and cancellation on current Android/iOS devices; record device/browser results in `specs/007-mobile-gameplay/quickstart.md`.

## Dependencies and Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: T001 must fail before T002 registers the new module route.
- **User Story 1 (Phase 3)**: Depends on setup; T005 follows T003, while T004 may run in parallel.
- **User Story 2 (Phase 4)**: Depends on setup and US1 shared-file work; tests T006-T008 precede implementation T009-T011.
- **US2 follow-up (Phase 5)**: Starts after US2 core work; T012-T013 precede T014.
- **User Story 3 (Phase 6)**: T015-T017 precede implementation; T018-T019 can proceed in parallel, T020 integrates both, and T021 can proceed alongside them.
- **Polish (Phase 7)**: T022 follows implementation; T023 requires real devices and follows T022.

### User Story Dependencies

- **User Story 1 (P1)**: Independent after setup.
- **User Story 2 (P1)**: Independent after US1; browser compatibility/orientation follow-up completes before results work.
- **User Story 3 (P1)**: Depends on the existing round-end protocol and final snapshot; no core scoring changes are introduced.

### Parallel Opportunities

- **US1**: T003 and T004 use different files; T005 follows T003.
- **US2**: T006-T008 test tasks use separate files; T009 and T010 may proceed after their tests, with T011 integrating.
- **US2 follow-up**: T012 and T013 use separate files and can run in parallel; T014 follows both.
- **US3**: T015-T017 use separate test files; T018 and T019 can run in parallel after tests, while T021 documentation can proceed independently.

## Implementation Strategy

The mobile-play MVP includes US1 and US2. Complete those first, then deliver results as an independent US3 increment. Keep tests-first sequencing, and finish with workspace quality gates and real-device checks where hardware is available.

## Format Validation

All tasks use `- [ ] T###`, include exact file paths, and carry `[US1]`, `[US2]` or `[US3]` only in their user-story phases. `[P]` appears only for independent work in different files.
