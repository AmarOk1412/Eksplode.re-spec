---
description: "Task list for controller and keyboard controls"
---

# Tasks: Controller and Keyboard Controls

**Input**: Design documents from `/specs/005-controller-keyboard-controls/`

**Prerequisites**: [plan.md](plan.md), [spec.md](spec.md), [research.md](research.md), [data-model.md](data-model.md), [contracts/control-settings.md](contracts/control-settings.md), [quickstart.md](quickstart.md)

**Tests**: Included because the project constitution requires test-first implementation and automated coverage of frontend input mappings and CLI behavior. Write each behavior test first and observe it fail before implementation.

**Organization**: Tasks are grouped by user story. Both stories are P1; the keyboard settings panel is scheduled first because the user's requested delivery order and the controller story's control panel depend on it.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel with another task because it edits independent files and has no dependency on incomplete work.
- **[Story]**: User story label, required for story-phase tasks.
- Every task names the file or files to change or validate.

## Path Conventions

- Rust workspace root: `game/`
- Web browser assets and tests: `game/crates/web/assets/`, `game/crates/web/js-tests/`, `game/crates/web/tests/`
- CLI sources and tests: `game/crates/cli/src/`, `game/crates/cli/tests/`
- User documentation: `game/README.md`, `game/docs/WEB.md`

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Make the new browser controls module available through the existing embedded-asset server.

- [X] T001 Add an HTTP regression test in `game/crates/web/tests/http.rs` asserting that `GET /controls.js` returns status 200, JavaScript content type, and the controls module body.
- [X] T002 Register `/controls.js` with `include_bytes!` in `game/crates/web/src/assets.rs` and create the initial ES module at `game/crates/web/assets/controls.js`; run `cargo test -p eksplode-re-web --test http` and confirm T001 passes.

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: No additional shared infrastructure is needed. T001-T002 establish the only new static-asset route; web and CLI changes can now proceed in their story phases.

---

## Phase 3: User Story 2 - Customize Keyboard Controls (Priority: P1) - MVP

**Goal**: Deliver an accessible web controls panel with persistent keyboard reassignment and reset, plus per-slot local CLI and single-player join CLI bindings.

**Independent Test**: Rebind all six web actions, use the bindings in a match, reload and verify persistence, restore defaults, then verify custom terminal bindings affect only the selected local slot or joining client. The controller need not be connected for this story to pass.

### Tests for User Story 2

> Write these tests first and observe them fail before implementing the behavior.

- [X] T003 [P] [US2] Add pure preference tests in `game/crates/web/js-tests/controls.test.mjs` for all default aliases, remapping, conflict/reserved-key rejection, versioned storage, invalid-value fallback, reset, and storage failure; extend `game/scripts/browser-smoke.mjs` to verify panel opening, key capture, conflict feedback, reset, and persistence across reload.
- [X] T004 [P] [US2] Add CLI argument tests in `game/crates/cli/tests/args.rs` for `--p1-keys` through `--p4-keys`, six-value `up,down,left,right,bomb,punch` order, `join --keys`, omitted-option defaults, case-insensitive characters, malformed values, duplicates, and reserved `q`, `p`, `r` keys.
- [X] T005 [P] [US2] Add mapping and integration tests in `game/crates/cli/tests/input.rs`, `game/crates/cli/tests/session.rs`, and `game/crates/cli/tests/online.rs` proving configured keys map to the requested action/slot, local gameplay uses the configured map, a joining client uses only its own map, and all defaults/command keys retain their meanings.

### Implementation for User Story 2

- [X] T006 [P] [US2] Implement the web keyboard preference model in `game/crates/web/assets/controls.js`: preserve aliases `ArrowUp`/`KeyW`, `ArrowDown`/`KeyS`, `ArrowLeft`/`KeyA`, `ArrowRight`/`KeyD`, `Space`/`KeyE`, and `KeyZ`/`KeyQ`; validate six actions, reject conflicts/reserved keys, and load/save `eksplode.re.controls.v1` with safe defaults on invalid data or storage errors.
- [X] T007 [US2] Update `game/crates/web/assets/app.js` to route keyboard events through the configured bindings while preserving held-direction priority, one-shot bomb/punch behavior, input-message shape, and clear-on-blur/hidden-page behavior.
- [X] T008 [P] [US2] Add an accessible controls-panel entry point and keyboard-binding/reset controls to `game/crates/web/assets/index.html`; include capture-state and status-message elements for conflict and reserved-key errors.
- [X] T009 [US2] Style the keyboard controls panel, capture state, error/status messages, and responsive layout in `game/crates/web/assets/style.css` without obscuring the game view; use the panel classes established in `game/crates/web/assets/index.html`.
- [X] T010 [US2] Wire panel open/close, key capture, displayed assignments, reset confirmation, persistence, and input suppression while editing in `game/crates/web/assets/app.js`; ensure changing bindings during play does not interrupt the match or leave an action held.
- [X] T011 [US2] Add parsing, validation, defaults, and help text for `--p1-keys` through `--p4-keys` and `join --keys` in `game/crates/cli/src/args.rs`; accept one character or `Up`, `Down`, `Left`, `Right`, `Space` per binding and reject duplicate or reserved keys with a usage error.
- [X] T012 [US2] Implement configured six-action terminal key maps in `game/crates/cli/src/input.rs`, retain existing mappings when no option is supplied, and preserve case-insensitive character handling and reserved command keys.
- [X] T013 [US2] Pass per-slot bindings from `LocalOptions` through `game/crates/cli/src/app.rs` and the single-player binding from `JoinOptions` through `game/crates/cli/src/online.rs`; verify a joining client can change only its own controls.
- [X] T014 [US2] Document web keyboard reassignment, persistence, reset, CLI key option syntax/order, accepted key names, collision errors, and unchanged defaults in `game/README.md` and `game/docs/WEB.md`.

**Checkpoint**: Keyboard settings work independently in the browser and terminal. The browser has a complete controls panel before controller integration begins.

---

## Phase 4: User Story 1 - Play with an Xbox-Compatible Controller (Priority: P1)

**Goal**: Add standard-mapped Xbox controller movement and actions through the existing web controls panel without requiring a keyboard.

**Independent Test**: Use synthetic standard-mapped gamepad states to verify D-pad/stick movement, dead zone, A-to-bomb, B-to-punch, one-shot edges, selection, disconnect and keyboard fallback; then verify the attached controller in a visible Linux browser.

### Tests for User Story 1

> Write these tests first and observe them fail before implementing the behavior.

- [X] T015 [US1] Add pure Gamepad mapping tests in `game/crates/web/js-tests/controls.test.mjs` for standard mapping validation, D-pad directions, left-stick directions with a 0.25 dead zone, Xbox A/B actions, one-shot button edges, unavailable/short controller state, and neutral state after disconnect or selection change.

### Implementation for User Story 1

- [X] T016 [US1] Implement a testable standard Gamepad mapper in `game/crates/web/assets/controls.js`: map D-pad and left stick to movement, button 0/A to bomb, button 1/B to punch, apply a 0.25 dead zone, and emit bomb/punch only on press edges.
- [X] T017 [US1] Integrate controller discovery, selection, fresh `navigator.getGamepads()` sampling and input arbitration into `game/crates/web/assets/app.js` on the existing 50 ms input cadence; handle missing/blocked API, focus/visibility loss, disconnect and device-selection changes by clearing held state and preserving keyboard fallback.
- [X] T018 [P] [US1] Extend the controller section in `game/crates/web/assets/index.html` to report availability, display Xbox assignments, prompt visible-page interaction when a connected device is not yet exposed, and let the player choose among multiple compatible devices.
- [X] T019 [US1] Extend `game/crates/web/assets/style.css` for controller status, selection and unavailable states using the controller markup in `game/crates/web/assets/index.html`, while preserving the keyboard panel's responsive layout.
- [X] T020 [US1] Document Linux/browser connection behavior, standard Xbox assignments, multiple-controller selection, keyboard fallback, and physical-device validation in `game/docs/WEB.md` and `game/README.md`.
- [ ] T021 [US1] Follow the Linux controller procedure in `specs/005-controller-keyboard-controls/quickstart.md` using the attached controller in a visible, focused browser; verify all directions, A/B actions, disconnect fallback, keyboard coexistence and no repeated action while a button is held, and record any browser-specific limitation in the quickstart.

**Checkpoint**: The selected Xbox-compatible controller can play a full match, and all keyboard settings remain usable.

---

## Phase 5: Polish & Cross-Cutting Concerns

**Purpose**: Run acceptance validation and repository quality gates across both control paths.

- [X] T022 Run `node --test crates/web/js-tests/*.test.mjs` from `game/` and verify web input mapping, storage, panel behavior and controller adapter tests all pass.
- [X] T023 Run `cargo fmt --check`, `cargo test --workspace`, and `cargo clippy --all-targets -- -D warnings` from `game/`; resolve any regressions in the touched CLI or web files.
- [X] T024 Run the browser smoke procedure in `game/scripts/browser-smoke.mjs` and the end-to-end scenarios in `specs/005-controller-keyboard-controls/quickstart.md`; verify no network protocol or game-rule changes were needed.

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: T001 before T002; the HTTP test establishes the embedded asset contract.
- **Foundational (Phase 2)**: No additional tasks; Phase 1 exposes the new shared web module.
- **User Story 2 (Phase 3)**: Starts after Phase 1; supplies the panel that the controller story extends.
- **User Story 1 (Phase 4)**: Starts after the keyboard panel in User Story 2; controller selection and status are added to that panel.
- **Polish (Phase 5)**: Depends on both stories and the Linux hardware check.

### User Story Dependencies

- **User Story 2 (P1)**: Depends only on Setup. Scheduled first to satisfy the requested panel-first delivery order and to establish keyboard settings.
- **User Story 1 (P1)**: Depends on User Story 2 because its controller status and selection controls extend the same web panel; gamepad mapping tests and adapter logic remain independently testable.

### Parallel Opportunities

- T003, T004 and T005 can be authored in parallel because they touch independent test files.
- After the US2 tests, T006, T008, T009 and T011 can proceed in parallel; T007/T010 depend on the web preference module and panel markup, while T012 depends on CLI option data.
- Within US1, T015 is the test-first prerequisite; afterward T016, T018 and T019 can proceed in parallel, then T017 integrates them.
- Documentation tasks can proceed in parallel with implementation in files they do not touch, but T020 shares documentation files with T014 and must follow it.

### Parallel Example: User Story 2

```text
T003 Web preference tests: game/crates/web/js-tests/controls.test.mjs
T004 CLI parser tests: game/crates/cli/tests/args.rs
T005 CLI mapping tests: game/crates/cli/tests/input.rs
```

### Parallel Example: User Story 1

```text
T016 Gamepad mapper: game/crates/web/assets/controls.js
T018 Controller panel markup: game/crates/web/assets/index.html
```

After T018 establishes the controller markup, T019 can style it in `game/crates/web/assets/style.css`.

## Implementation Strategy

### MVP First: User Story 2

1. Establish the embedded asset route.
2. Write and fail the keyboard preference and CLI mapping tests.
3. Implement the browser keyboard settings panel and the CLI per-run key options.
4. Validate persistence/reset in the browser and independent CLI slot/join behavior.
5. Stop at the US2 checkpoint; keyboard customization is usable before controller support.

### Incremental Delivery

1. Complete US2 and validate browser keyboard remapping and CLI bindings.
2. Add US1 Gamepad mapping, controller selection and status in the existing panel.
3. Verify with synthetic tests, then the attached controller in a visible Linux browser.
4. Run workspace tests, formatting, clippy and the browser smoke scenario.

## Notes

- Task IDs are sequential; all tasks use checkboxes and exact paths.
- `[P]` appears only where file ownership and dependencies permit parallel work.
- `[US1]` maps to the controller story; `[US2]` maps to keyboard customization.
- Tests are required by the project constitution and must fail before their corresponding implementation.
- The feature changes presentation input only; the server protocol, game rules and core crate remain unchanged.
- T021 physical-device validation remains open until native Chromium exposes the attached controller through the Gamepad API.

## Phase 6: Convergence

- [ ] T025 Add and document a per-run key-binding option for the terminal host's own player, parse it in `game/crates/cli/src/args.rs`, pass it through `HostOptions` and `game/crates/cli/src/app.rs` to `Online::hosting` in `game/crates/cli/src/online.rs`, and cover parsing and host input in `game/crates/cli/tests/args.rs` and `game/crates/cli/tests/online.rs` per FR-008 and plan: terminal per-run bindings (partial).
- [ ] T026 Reject Ctrl/Alt/Meta-modified shortcut chords during web key capture in `game/crates/web/assets/app.js`, preserve the prior binding with reserved-key feedback, and cover the behavior in `game/scripts/browser-smoke.mjs` per FR-009 and the reserved/browser-shortcut edge case (partial).
- [ ] T027 Complete physical validation with the attached Xbox-compatible controller in a visible, focused Linux browser; verify browser exposure, movement, A/B actions, disconnect fallback, and held-action safety, and diagnose or document any remaining browser/device limitation in `specs/005-controller-keyboard-controls/quickstart.md` per US1/AC1, US1/AC3, and SC-001 (partial).