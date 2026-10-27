---
description: "Task list for mouse-free game navigation"
---

# Tasks: Mouse-Free Game Navigation

**Input**: Design documents from `/specs/006-mouse-free-game-navigation/`

**Prerequisites**: [plan.md](plan.md), [spec.md](spec.md), [research.md](research.md), [data-model.md](data-model.md), [contracts/navigation-controls.md](contracts/navigation-controls.md), [quickstart.md](quickstart.md)

**Tests**: Included because the project constitution requires test-first work and tested input mapping for frontends. Write each test first and watch it fail before the code it covers.

**Organization**: The requested delivery order is by screen area: settings, then the lobby, then the main screen. Each area is a phase and is complete for both input methods. Tasks are labeled `[US1]` (keyboard, spec User Story 1) or `[US2]` (controller, spec User Story 2); shared markup and styles carry `[US1]` because the keyboard path is the baseline.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependency on unfinished tasks).
- **[Story]**: `[US1]` keyboard or `[US2]` controller; only on the screen-area phases.
- Every task names its file or files.

## Path Conventions

- Rust workspace root: `game/`
- Web assets, JavaScript tests and server tests: `game/crates/web/assets/`, `game/crates/web/js-tests/`, `game/crates/web/tests/`, `game/crates/web/src/`
- Smoke script and docs: `game/scripts/browser-smoke.mjs`, `game/README.md`, `game/docs/WEB.md`

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Serve the new navigation module from the embedded-asset server.

- [X] T001 Add `/nav.js` with content type `text/javascript; charset=utf-8` to the asset lists in `game/crates/web/tests/http.rs` (the route test and the page-size test) and run `cargo test -p eksplode-re-web --test http` to see it fail with 404.
- [X] T002 Register `/nav.js` with `include_bytes!` in `game/crates/web/src/assets.rs`, create the module shell `game/crates/web/assets/nav.js`, and confirm T001 passes.

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: The shared navigation logic and wiring that every screen area uses.

**CRITICAL**: No screen-area work starts until this phase is complete.

### Tests first (separate files, so they can be written in parallel)

- [X] T003 [P] Write `game/crates/web/js-tests/nav-focus.test.mjs` for direction movement over fake elements and rectangles: the nearest enabled target in the pressed direction wins, DOM order is the fallback when none lies in that direction, and a focus target is "Visible, not disabled, not inert, with a non-empty bounding rectangle"; include a stable-key helper that remembers and restores the focused element after a re-render, and a stable result when no target exists.
- [X] T004 [P] Write `game/crates/web/js-tests/nav-layers.test.mjs` for the layer stack: ids `hub`, `sound`, `controls`, `osk` with "at most one of each open", the opener is recorded, only the top layer is active, closing restores focus to the opener if it is still enabled, else to the first enabled control of the layer below, and back closes only the top layer.
- [X] T005 [P] Write `game/crates/web/js-tests/nav-input.test.mjs` for the controller navigation tracker and routing: D-pad and left stick, the stick "presses at 0.5, releases at 0.3", direction repeat "at 400 ms then every 120 ms while held", A, B and Start fire once, every button must be released once after a context change, a missing controller or short arrays count as neutral, and the route is `gameplay` only when the stage is `playing` and no layer is open.
- [X] T006 [P] Write `game/crates/web/js-tests/nav-adjust.test.mjs` for adjustable controls: left and right on a select cycle the value and fire `change`, on a range they step within min, max and step and fire `input` then `change`, on a radio group they move the selection, A toggles a checkbox and presses a button, A on a text field requests the on-screen keyboard, and up and down always leave the control.

### Implementation

- [X] T007 Implement direction movement, the focus-target filter and the focus-key restore helper in `game/crates/web/assets/nav.js` until T003 passes.
- [X] T008 Implement the layer stack in `game/crates/web/assets/nav.js` until T004 passes.
- [X] T009 Implement the controller navigation tracker and the navigation-or-gameplay routing in `game/crates/web/assets/nav.js` until T005 passes.
- [X] T010 Implement the adjustable-control helpers in `game/crates/web/assets/nav.js` until T006 passes.
- [X] T011 [P] Add a visible focus indicator for every control type (text fields, selects, range sliders, checkboxes, mode buttons, colour swatches, slot buttons) in `game/crates/web/assets/style.css`; today only `button:focus-visible` is styled.
- [X] T012 Wire the shared behavior in `game/crates/web/assets/app.js`: sample the controller on the existing 50 ms tick in every stage and route the result with the T009 router, keep held gameplay input cleared while in navigation, remember and restore the focused control across `updateUi` re-renders, and blur the focused control when the stage becomes `playing` so Space and Enter cannot press a hidden button.

**Checkpoint**: The pure navigation logic is tested and the page routes input; screen areas can now be built in order.

---

## Phase 3: Settings (Keyboard and Controller)

**Goal**: A menu hub plus the sound and controls panels operable without a mouse, reachable from every screen.

**Independent Test**: Open the hub with Escape or Start, adjust both sound sliders and Mute, rebind a key, cancel a capture with Escape, restore defaults with confirmation, and close everything, with focus returning to the opener each time and no bomb sent while a layer is open.

### Tests for this phase

- [X] T013 [P] [US1] Add markup assertions to `game/crates/web/tests/http.rs` for the hub (`menu-hub`, `hub-sound`, `hub-controls`, `hub-resume`) and dialog roles with labels on the sound and controls panels.
- [X] T014 [US1] Extend `game/scripts/browser-smoke.mjs` with a keyboard-only settings walk: Escape opens the hub, Sound opens, ArrowRight changes a slider, Escape returns to the hub, Controls opens, a key capture is cancelled with Escape and leaves the binding unchanged, restore defaults needs confirmation, and focus returns to each opener.
- [X] T015 [US2] Extend `game/scripts/browser-smoke.mjs` with a synthetic-controller settings walk using the existing fake Gamepad harness: Start opens the hub, D-pad moves focus, left and right adjust a slider, A toggles Mute, B goes back, the controller selector cycles, and no `Input` message with bomb or punch is sent while a layer is open.

### Implementation for this phase

- [X] T016 [US1] Add the menu hub layer (Sound, Controls, Resume) and dialog roles and labels to the sound and controls panels in `game/crates/web/assets/index.html`.
- [X] T017 [US1] Style the hub layer in `game/crates/web/assets/style.css` using the classes from T016 so it never covers the game canvas controls it opens over.
- [X] T018 [US1] In `game/crates/web/assets/app.js` make the hub, sound and controls panels layers: Escape opens the hub when no layer is open and closes the top layer otherwise, everything outside the top layer is `inert`, focus returns to the opener on close, and during controls key capture Escape cancels the capture instead of being reported as reserved.
- [X] T019 [US2] In `game/crates/web/assets/app.js` drive the settings layers from the controller: Start opens and closes the hub, B closes the top layer, D-pad and stick move focus, A activates, left and right adjust the sliders and cycle the controller selector, and the release-first rule applies when a layer opens or closes.

**Checkpoint**: Settings are fully mouse-free with both inputs; T014 and T015 pass.

---

## Phase 4: Lobby (Keyboard and Controller)

**Goal**: Room settings, slot actions, the character editor and text entry operable without a mouse.

**Independent Test**: From the lobby, rename the room, change mode, board and time, add a bot, open Change look, pick a colour, press Done, mark Ready and start, using only the keyboard and then only the controller.

### Tests for this phase

- [X] T020 [P] [US2] Write `game/crates/web/js-tests/osk.test.mjs` for the on-screen keyboard reducer: the buffer is "Never longer than the field's `maxlength` (player name 16, room name 24); control characters are dropped", shift changes letter case, backspace and space work, confirm returns the buffer, cancel restores the original, the cursor stays on an enabled key, and an empty player name is allowed in the buffer.
- [X] T021 [P] [US1] Add markup assertions to `game/crates/web/tests/http.rs` for the on-screen keyboard layer (`osk` and its key grid) and for the lobby controls staying reachable.
- [X] T022 [US1] Extend `game/scripts/browser-smoke.mjs` with a keyboard-only lobby walk: Tab to the room name and type, change mode and board with the keyboard, Enter on a focused Add bot adds a bot and does not start the match, Change look moves focus into the editor, a colour is chosen with arrows, Done returns focus to Change look, Ready toggles, and Enter on Start starts.
- [X] T023 [US2] Extend `game/scripts/browser-smoke.mjs` with a synthetic-controller lobby walk: rename the room with the on-screen keyboard (confirm and cancel), cycle mode, board and time with left and right, A on Add bot, enter and leave the character editor.

### Implementation for this phase

- [X] T024 [US2] Implement the on-screen keyboard reducer in `game/crates/web/assets/nav.js` until T020 passes.
- [X] T025 [US2] Add the on-screen keyboard layer markup (letters, digits, space, a few punctuation marks, shift, backspace, confirm, cancel) to `game/crates/web/assets/index.html`.
- [X] T026 [US2] Style the on-screen keyboard and the lobby controls' focus states in `game/crates/web/assets/style.css`.
- [X] T027 [US1] In `game/crates/web/assets/app.js` make the lobby keyboard-operable: the window Enter-to-start handler must apply only when the target is not a button, focus enters the character editor when Change look opens it and returns to Change look on Done, lobby re-renders (slot cards) keep focus, arrows move between lobby buttons using the T007 movement, and colour swatches move their selection with left and right.
- [X] T028 [US2] In `game/crates/web/assets/app.js` drive the lobby from the controller and open the on-screen keyboard from a focused text field with A (confirm writes the text back and fires `input`, cancel or B restores the original and returns focus to the field), with left and right adjusting the lobby selects, checkbox and the editor selects.

**Checkpoint**: The lobby is fully mouse-free with both inputs; T022 and T023 pass.

---

## Phase 5: Main Screen (Keyboard and Controller)

**Goal**: Room creation, joining and the running match operable without a mouse.

**Independent Test**: From a fresh page load, create a room, enter a player name, join, start, open the hub during the match without moving or bombing, resume and return to the lobby, with the keyboard only and then the controller only.

### Tests for this phase

- [X] T029 [US1] Extend `game/scripts/browser-smoke.mjs` with a full keyboard-only flow from a fresh load: room mode choice with arrows, Create room, player name, Join, add bot, Start, Escape opens the hub in the match with no `Input` message carrying movement or bomb while it is open, Resume, play, and return to the lobby.
- [X] T030 [US2] Extend `game/scripts/browser-smoke.mjs` with the same full flow using only the synthetic controller, including player-name entry through the on-screen keyboard and Start opening the hub during the match.

### Implementation for this phase

- [X] T031 [US1] In `game/crates/web/assets/app.js` make room creation and joining keyboard-operable: the mode choices move with left and right and keep focus when `renderRoomMenu` rebuilds them, Enter in the name fields keeps its current meaning, initial focus lands on the first enabled control on load and on every stage change, and the closed state focuses Retry.
- [X] T032 [US1] In `game/crates/web/assets/app.js` make the match safe: Escape opens the hub while playing, gameplay input is suppressed and held input cleared while any layer is open, and closing the last layer resumes play without a stuck or repeated action; on return to the lobby focus goes to the first lobby control.
- [X] T033 [US2] In `game/crates/web/assets/app.js` drive room creation, joining (player name through the on-screen keyboard) and the match from the controller, with Start opening the hub during play and the feature 005 gameplay mapping unchanged when no layer is open.
- [X] T034 [US1] In `game/crates/web/assets/app.js` and `game/crates/web/assets/index.html` make the developer panel (pause, reset, the drop sliders) reachable and operable by keyboard and controller when it is shown, and skipped by navigation when hidden; verify against a server started with `--dev`.

**Checkpoint**: The whole web flow is mouse-free with both inputs; T029 and T030 pass.

---

## Phase 6: Polish, Validation and Commit

**Purpose**: Documentation, quality gates, hardware check and the final commit.

- [X] T035 [P] Document keyboard navigation, controller menu navigation, controller text entry, the menu hub and mouse-free access to every screen in `game/README.md` and `game/docs/WEB.md`; read the current contents first because earlier edits to these files were reverted, and keep the options table and links valid.
- [X] T036 Run `node --test crates/web/js-tests/*.test.mjs` from `game/` and fix any failure.
- [X] T037 Run `cargo fmt --check`, `cargo clippy --all-targets -- -D warnings` and `cargo test --workspace` from `game/`, including `cargo test -p eksplode-re-web --test docs`, and fix any failure.
- [X] T038 Run `node scripts/browser-smoke.mjs` against a fresh server as described in [quickstart.md](quickstart.md) and confirm a clean console and all keyboard and controller walks pass.
- [X] T039 Repeat the manual checks in [quickstart.md](quickstart.md) with the attached Xbox-compatible controller in a visible, focused Linux browser; if the browser still lists no controller, diagnose with `jstest-gtk` and record the limitation in `specs/006-mouse-free-game-navigation/quickstart.md` as feature 005 did. Physical validation was blocked: `jstest-gtk` is installed/launched, but no `/dev/input/js*` device or Xbox/gamepad/joystick kernel entry is present; the limitation is recorded in `quickstart.md`.
- [X] T040 Commit the work from `game/`: reviewed `git status` and `git diff`, staged explicit paths (never `git add -A`, and left unrelated changes such as `docs/DEVELOPMENT.md` out), and made separate implementation and documentation commits with subjects accepted by `.githooks/commit-msg`; did not use `--no-verify` or push.

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: T001 before T002.
- **Foundational (Phase 2)**: After Phase 1. T003 to T006 first (parallel); T007 to T010 follow their tests in the same file, in order; T011 is independent; T012 needs T007 to T010.
- **Settings (Phase 3)**: After Phase 2. T016 before T017 and T018; T018 before T019.
- **Lobby (Phase 4)**: After Phase 3, because it reuses the layer wiring and hub; T024 before T028; T025 before T026; T027 and T028 need T018.
- **Main screen (Phase 5)**: After Phase 4, because join needs the on-screen keyboard.
- **Polish (Phase 6)**: After all phases; T040 last.

### Why the screen phases are sequential

All screen-area wiring edits `game/crates/web/assets/app.js`, and the smoke walks share `game/scripts/browser-smoke.mjs`, so the phases follow the requested order rather than running side by side.

### Parallel Opportunities

- T003, T004, T005 and T006 (separate test files) can be written together.
- T011 (`style.css`) can run alongside T007 to T010 (`nav.js`).
- T013 (`http.rs`) can run alongside T016 (`index.html`) only after its markup contract is agreed; it is listed first for test-first order.
- T020 (`osk.test.mjs`) and T021 (`http.rs`) are independent.
- T035 (docs) can run alongside T036 to T038.

### Parallel Example: Foundational Tests

```text
T003 Focus movement tests: game/crates/web/js-tests/nav-focus.test.mjs
T004 Layer stack tests:    game/crates/web/js-tests/nav-layers.test.mjs
T005 Controller input tests: game/crates/web/js-tests/nav-input.test.mjs
T006 Adjustable control tests: game/crates/web/js-tests/nav-adjust.test.mjs
```

## Implementation Strategy

### MVP First: Settings

1. Complete Setup and Foundational.
2. Complete the Settings phase; stop and validate T014 and T015. Settings and the menu hub are usable with the keyboard and controller before any other screen changes.

### Incremental Delivery

1. Settings, then validate.
2. Lobby (adds the on-screen keyboard), then validate.
3. Main screen (room creation, join, match), then validate the full flow.
4. Polish, hardware check, quality gates and commit.

## Notes

- Task IDs are sequential; `[P]` marks different files with no dependency on unfinished tasks.
- Tests are written first and must fail before their code.
- The feature is presentation-only: no change to the server protocol, game rules, core crate or terminal client.
- Feature 005's physical controller check (its T027) is still open; T039 depends on the same device exposure.
- `specs/` is outside the `game/` repository, so this file is not part of the commit in T040.
