# Implementation Plan: Mouse-Free Game Navigation

**Branch**: `006-mouse-free-game-navigation` | **Date**: 2026-10-27 | **Spec**: [spec.md](spec.md)

**Input**: Feature specification from `/specs/006-mouse-free-game-navigation/spec.md`. User direction: deliver mouse-free control in this order: settings, then the lobby, then the main screen (room creation, join, match).

## Summary

Make every web screen operable with the keyboard and the selected Xbox-compatible controller. One focus model drives both: real DOM focus, moved by Tab or arrows for the keyboard and by the D-pad or left stick for the controller, with A to activate, B or Escape to go back, and Start or Escape to open a menu hub (Sound, Controls, Resume) from any screen. A layer stack traps focus in the open panel and restores it to the opener. Controller text entry uses an on-screen keyboard built from real buttons. Delivery follows the requested order: first the settings panels and the menu hub, then the lobby (room settings, slots, character editor, on-screen keyboard), then the main screen (room creation, join, in-match menu, developer panel). Presentation only: no game rules, protocol or core changes.

## Technical Context

**Language/Version**: Native JavaScript ES modules in the browser; Rust stable (edition 2021) only for the embedded-asset route.

**Primary Dependencies**: None new. Browser Gamepad API (already used by feature 005) and the Node.js built-in test runner.

**Storage**: N/A. Navigation state is runtime-only and nothing new is persisted.

**Testing**: `node --test crates/web/js-tests/*.test.mjs` with fake elements, rectangles and Gamepad states; `cargo test -p eksplode-re-web --test http`; the CDP smoke script `scripts/browser-smoke.mjs` for keyboard-only and synthetic-controller walks; a physical controller check as in feature 005.

**Target Platform**: Linux-first web client in current Chromium and Firefox; other platforms keep working through the keyboard.

**Project Type**: Rust workspace with a presentation-only JavaScript client served by `crates/web`.

**Performance Goals**: Controller navigation samples on the existing 50 ms tick; no new animation loop. A focus move shows within one tick of the input.

**Constraints**: Hidden and disabled controls are never focus targets. Navigation input never produces bomb, punch or movement messages. Buttons held across a context change are ignored until released. Controller access stays optional and the keyboard path works without it. Standard mapping only: A 0, B 1, Start 9, D-pad 12 to 15, left stick axes 0 and 1.

**Scale/Scope**: Seven web views (room creation, join, character editor, lobby, match, sound panel, controls panel), two new layers (menu hub, on-screen keyboard) and the optional developer panel.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

- **Specification-first**: PASS. Work traces to FR-001 to FR-012 in the approved spec.
- **Test-first and coverage**: PASS. Pure focus, layer, controller-navigation and on-screen-keyboard logic gets Node tests written first; each acceptance scenario maps to a Node test or a smoke step.
- **Core/frontend separation**: PASS. All changes are in `crates/web/assets/` plus one asset route; no rules move into the browser.
- **Deterministic simulation**: PASS. No simulation, tick or RNG change.
- **Four-player multiplayer with bots**: PASS. Input messages and slots are unchanged.
- **Dual-interface parity**: PASS. The terminal client is already fully keyboard-driven (setup screen, lobby keys, game keys); this feature brings the web client to the same level.
- **Technology and platform**: PASS. No dependency, no `unsafe`, no CSP change; presentation-only code under `crates/web/assets/`.
- **Documentation and gates**: PASS. README and `docs/WEB.md` get a navigation section; `cargo fmt --check`, `cargo clippy --all-targets -- -D warnings` and `cargo test --workspace` run at the end.
- **Post-design Constitution Check**: PASS. The design adds one presentation module and no rule, protocol or dependency change.

## Project Structure

### Documentation (this feature)

```text
specs/006-mouse-free-game-navigation/
├── plan.md              # This file (/speckit-plan command output)
├── research.md          # Phase 0 output (/speckit-plan command)
├── data-model.md        # Phase 1 output (/speckit-plan command)
├── quickstart.md        # Phase 1 output (/speckit-plan command)
├── contracts/           # Phase 1 output (/speckit-plan command)
└── tasks.md             # Phase 2 output (/speckit-tasks command - NOT created by /speckit-plan)
```

### Source Code (repository root)

```text
game/crates/web/assets/
├── nav.js                 # New: focus movement, layer stack, controller navigation, on-screen keyboard logic
├── app.js                 # Wire layers, polling in every stage, focus restore on re-render, Escape/Start hub
├── index.html             # Menu hub and on-screen keyboard layers, dialog roles
└── style.css              # Visible focus for every control type, hub and on-screen keyboard styles
game/crates/web/src/assets.rs   # Embedded route for /nav.js
game/crates/web/js-tests/       # nav.test.mjs: pure navigation tests
game/crates/web/tests/http.rs   # Route and markup assertions
game/scripts/browser-smoke.mjs  # Keyboard-only and synthetic-controller walks
game/README.md and game/docs/WEB.md  # Navigation documentation
```

**Structure Decision**: Keep all navigation logic in one new presentation module, `nav.js`, imported by `app.js`, mirroring how `controls.js` was added by feature 005. Wiring stays in `app.js`. No Rust logic changes beyond registering the asset.

**Delivery order (requested)**: (1) Settings: shared navigation core, menu hub, sound panel, controls panel. (2) Lobby: room settings, slots and buttons, character editor, on-screen keyboard for room and player names. (3) Main screen: room creation menu, join screen, in-match menu, developer panel, retry and closed states.

**Documentation note**: The README and `docs/WEB.md` edits made for feature 005 were reverted by the user, so the navigation documentation task must read the current files and not assume that text exists.

## Complexity Tracking

No constitution violations.
