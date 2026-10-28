# Implementation Plan: Mobile Browser Gameplay

**Branch**: `007-mobile-gameplay` | **Date**: 2026-10-27 | **Spec**: [spec.md](spec.md)

**Input**: Feature specification from `/specs/007-mobile-gameplay/spec.md`

**Note**: The implementation task list is generated separately by `/speckit-tasks` after this plan is approved.

## Summary

Add phone touch controls with a portrait layout that places the joystick bottom-left and actions bottom-right, while preserving the existing landscape layout. Detect touch by capabilities so Firefox for Android works without browser-name exceptions. Show Punch only while the player has that power. After a round ends, retain the winner/draw final-standings screen until Return to Lobby is selected, without changing game rules or network messages.

## Technical Context

**Language/Version**: Browser JavaScript ES modules; Rust stable, edition 2021 for the web server and embedded asset registry.

**Primary Dependencies**: No new packages. Use browser Pointer Events, `matchMedia`, and the existing Node.js built-in test runner.

**Storage**: N/A. Touch gesture and orientation state are transient and remain in the browser page.

**Testing**: Node tests for touch capability, portrait/landscape layout and conditional Punch in `game/crates/web/js-tests/`; Rust markup/asset tests in `game/crates/web/tests/http.rs`; CDP browser smoke in `game/scripts/browser-smoke.mjs`; manual checks on iOS Safari, Firefox for Android and Android Chrome.

**Target Platform**: Current iOS Safari, Firefox for Android and other mainstream iOS/Android browsers, plus existing desktop browsers; Linux-first server and development environment.

**Project Type**: Rust workspace with a server-served web client; presentation-only browser code is under `game/crates/web/assets/`.

**Performance Goals**: Touch input enters the existing 20 Hz gameplay-input cadence; no extra polling or animation loop. The on-screen joystick responds visually to the current touch.

**Constraints**: Support both orientations; portrait uses bottom-corner controls and landscape preserves the existing layout. Clear held input on cancellation, focus loss and orientation changes. Punch is hidden unless the player owns that power. Preserve keyboard, mouse, controller, game rules and protocol. Results persist until explicitly dismissed. No dependencies or unsafe code.

**Scale/Scope**: One touch-controlled browser player per page in the existing four-participant match. Changes are limited to match presentation, input mapping, embedded asset registration, tests and user documentation.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

- **Specification-first**: PASS. Work traces to FR-001 through FR-017 and the three user stories.
- **Test-first and coverage**: PASS. Capability detection, orientation changes, conditional Punch and result persistence have focused tests; physical-device checks are explicitly recorded when unavailable.
- **Core/frontend separation**: PASS. Browser code only maps input and presents outcomes; game rules and scoring remain unchanged.
- **Deterministic simulation**: PASS. No simulation, clock, RNG or tick behavior changes.
- **Four-player multiplayer with bots**: PASS. Participant slots, server authority and shared outcomes remain unchanged.
- **Dual-interface parity**: PASS. Mobile behavior extends the web client; desktop and terminal controls remain available.
- **Technology and platform**: PASS. No new dependencies or unsafe code; all UI remains in existing web assets.
- **Documentation and gates**: PASS. README/WEB guide changes and workspace quality gates are included.
- **Post-design Constitution Check**: PASS. No core, protocol, or game-rule changes are planned.

## Project Structure
### Documentation (this feature)

```text
specs/007-mobile-gameplay/
├── plan.md
├── research.md
├── data-model.md
├── quickstart.md
└── contracts/touch-controls.md
```

### Source Code (repository root)

```text
game/crates/web/assets/
├── touch.js                 # Pure joystick gesture mapping and touch action state
├── app.js                   # Integrate touch/orientation with the existing input tick
├── index.html               # Match-only portrait/landscape controls and results view
└── style.css                # Portrait bottom-corner and landscape touch/result layouts
game/crates/web/src/assets.rs       # Register the embedded /touch.js asset
game/crates/web/tests/http.rs       # Assert asset route and match markup
game/crates/web/js-tests/touch.test.mjs  # Joystick, multi-touch and cancellation tests
game/scripts/browser-smoke.mjs      # Mobile viewport and touch smoke flow
game/README.md                      # User-facing control summary
game/docs/WEB.md                    # Mobile browser and control guidance
```

**Structure Decision**: Keep input normalization and capability helpers in `touch.js`, following the existing `controls.js` and `nav.js` pattern. `app.js` owns orientation-responsive control visibility, conditional Punch display, result retention and touch integration with the existing input tick. No core, net, CLI, renderer or protocol changes are planned.

## Complexity Tracking

No Constitution violations.
