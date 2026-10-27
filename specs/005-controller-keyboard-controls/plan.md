# Implementation Plan: Controller and Keyboard Controls

**Branch**: `005-controller-keyboard-controls` | **Date**: 2026-10-27 | **Spec**: [spec.md](spec.md)

**Input**: Feature specification from `/specs/005-controller-keyboard-controls/spec.md`. User direction: ship a controls configuration panel first, then add Xbox controller support within that panel; development and hardware validation target Linux with an Xbox-compatible controller attached.

## Summary

Add a dedicated web controls panel for keyboard inspection, reassignment, reset and controller selection. Deliver the panel and persistent keyboard remapping first, then add standard-mapped controller input through the same panel. Keep input translation in the presentation clients: the web client and CLI produce the existing movement, bomb and punch actions, leaving game rules, server authority and the network protocol unchanged. The CLI exposes per-slot keyboard bindings for the current run.

## Technical Context

**Language/Version**: Rust stable, workspace edition 2021; browser client uses native JavaScript modules.

**Primary Dependencies**: Existing Cargo workspace, browser Gamepad API, and Node.js built-in test runner; no new runtime dependency planned.

**Storage**: Versioned browser `localStorage` preferences for web keyboard bindings; terminal key bindings are supplied for the current CLI run and are not persisted.

**Testing**: `node --test crates/web/js-tests/*.test.mjs`; focused CLI tests and `cargo test --workspace`; interactive browser test with the attached controller on Linux.

**Target Platform**: Linux-first web game served locally, browser client, and terminal client; other supported platforms must retain keyboard operation.

**Project Type**: Rust workspace with CLI and web server, plus a presentation-only JavaScript browser client and WebAssembly renderer.

**Performance Goals**: Inputs continue to reach the existing 20 Hz game input loop without perceptible delay; controller polling must not create a separate continuously running animation loop.

**Constraints**: Preserve existing keyboard defaults; reject invalid/conflicting mappings; clear held state on blur, disconnect and selection changes; controller support requires a browser-exposed standard mapping; no game-rule, protocol or core-crate changes.

**Scale/Scope**: Up to four local terminal player slots; one web player per browser client; multiple connected controllers may be listed, with one selected for that client.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

- **Specification-first**: PASS. Work is bounded by the approved feature specification; the plan records a clarifying scope split for persistent web preferences versus per-run terminal bindings.
- **Test-first and coverage**: PASS. Add pure mapping/storage tests and CLI parser/mapping tests before implementation; exercise UI and physical controller scenarios after integration.
- **Core/frontend separation**: PASS. Control devices are translated in frontend clients into existing game actions; no input-device logic enters `core`.
- **Deterministic simulation**: PASS. Device polling only constructs the ordered inputs supplied to the existing deterministic tick; it introduces no simulation randomness or wall-clock rule.
- **Four-player parity**: PASS. Existing CLI multi-slot mappings remain the defaults; each local terminal slot can be configured independently. Web remains one player per browser.
- **Dual-interface parity**: PASS. Both web and terminal support keyboard remapping; controller hardware support is web-only and does not change the game rules.
- **Platform, dependencies and security**: PASS. Use the browser's built-in Gamepad API and standard gamepad mapping; no dependency or CSP exception is needed. A missing, blocked or non-standard controller falls back to keyboard with a status message.
- **Documentation and gates**: PASS. Update web and CLI control documentation; run relevant JS and Rust tests, then the workspace quality gates at implementation completion.
- **Post-design Constitution Check**: PASS. The selected asset, CLI and test boundaries preserve frontend ownership, existing input contracts, deterministic rules, minimal dependencies and both-interface keyboard support; no gate changed during research or design.

## Project Structure

### Documentation (this feature)

```text
specs/005-controller-keyboard-controls/
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
├── index.html             # Controls entry point and configuration panel
├── app.js                 # Existing client state and 20 Hz input dispatch
├── controls.js            # Keyboard preferences, controller discovery and input mapping
└── style.css              # Controls panel presentation
game/crates/web/src/assets.rs  # Embedded route for the controls module
game/crates/web/js-tests/      # Pure mapping, validation, storage and UI-state tests
game/crates/cli/src/
├── args.rs                # Per-slot key-binding options and validation
├── input.rs               # Default and configured terminal key mapping
├── app.rs                 # Apply configured bindings to local play
└── online.rs              # Apply configured bindings to terminal join play
game/crates/cli/tests/         # CLI options, mapping and session tests
game/README.md and game/docs/WEB.md  # Terminal and web controls documentation
```

**Structure Decision**: Keep browser-specific settings and controller translation in a small presentation module imported by the existing page. Register that module in the web server's fixed embedded-asset table. Keep terminal key parsing and mapping in the existing CLI input boundary. Do not add a renderer, core, bot, network or protocol module for device input.

## Complexity Tracking

> **Fill ONLY if Constitution Check has violations that must be justified**

No constitution violations.
