# Implementation Plan: Animated Lobby Scene

**Branch**: `main` (actual Git branch, unchanged) | **Date**: 2026-11-06 | **Spec**: [spec.md](spec.md)

**Input**: Feature specification from `specs/014-lobby-animation/spec.md`, including the planning refinement and confirmed main-menu placement.

**Feature identifier**: `014-lobby-animation`. The setup script reports this as `BRANCH`; `git branch --show-current` reports `main`. No branch or worktree is created by this planning command.

## Summary

**User revision (2026-11-06)**: Remove the scene pause/play icon and its UI wiring. Preserve reduced-motion support and automatic suspension when hidden or covered. Earlier pause-control design details below are superseded by this revision; internal controller pausing may remain for deterministic test fixtures, not as a player-facing control.

Replace the existing main-menu `#logo` zone, not the room lobby, with an unframed 2D pixel-art scene. Randomly select short vignettes showing running bomb drops, foreground/background encounters, karts, cartoon explosion endings, and a bomb-push/flattening gag. Preserve the existing game heading, name input, Create new game, Join a game, Customize, Options, and retry behavior.

Implement a pure, seeded decorative timeline and drawing-command builder in Rust `web-render`. Reuse directional character sprites, bomb colours, and the existing 16-float command ABI. Browser JavaScript manages a single animation loop, injected ticks, sizing, visibility, pause, and reduced motion; it neither chooses choreography nor decides game rules. A same-origin PNG supplies a non-WASM fallback. No match model, room protocol, terminal behavior, audio, or third-party dependency changes.

## Technical Context

**Language/Version**: Rust 2021, workspace minimum Rust 1.75, stable toolchain; existing browser ES modules, HTML, and CSS.

**Primary Dependencies**: Existing `eksplode-re-core::Rng`, `web-render::look`, `sprites`, `commands`, and WASM FFI; browser Canvas 2D, requestAnimationFrame, matchMedia, and Web Crypto. No added package or framework.

**Storage**: In-memory timeline and pause choice for the current page visit; no new local storage, server storage, migrations, or network messages.

**Testing**: Native `cargo test` for timeline/render/FFI; Node's existing test runner for browser glue, focus, lifecycle, and reduced-motion behavior; web HTTP asset tests; browser automation and screenshots for real WASM rendering, mobile layout, and interactions; a five-player review for SC-006.

**Target Platform**: Existing Linux-hosted server and supported desktop/mobile browsers; `wasm32-unknown-unknown` renderer. Terminal client unchanged.

**Project Type**: Presentation-only enhancement to an existing Rust workspace web client.

**Performance Goals**: First moving character within one second of the menu becoming usable, including graphics startup latency rather than excluding it; one scene RAF at most, none while inactive; target 60 fps on normal desktop/mobile hardware; all required motifs within every 90 seconds of active playback; no empty transition over two seconds.

**Constraints**: Existing WASM limit 307,200 bytes and total page asset limit 512,000 bytes, including new fallback; fixed scene dimensions with responsive constraints; silent, non-flashing cartoon effects; no new unsafe code outside the existing FFI boundary; no room/game state mutations.

**Measured Baseline**: Existing embedded files total 435,777 bytes, including 240,384 bytes of WASM; remaining aggregate headroom is 76,223 bytes and WASM headroom is 66,816 bytes. These are pre-implementation filesystem measurements, not delivered feature sizes.

**Scale/Scope**: One main-menu scene, seven five-second vignette definitions, up to four decorative actors and three bombs per frame, one static fallback and one pause/resume icon button. These are presentation limits, not match capacity or gameplay rules.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principle or Gate | Pre-research Assessment | Post-design Assessment |
|-------------------|-------------------------|------------------------|
| I. Specification-first | PASS: refined spec records user-confirmed placement and new motifs before implementation | PASS: every design surface traces to FR-001 through FR-015; planning does not implement code |
| II. Test-first and coverage | PASS: require failing timeline/FFI/glue tests before code | PASS: acceptance matrix in quickstart assigns all scenarios automated coverage, plus visual smoke and human review |
| III. Core/frontend separation | PASS: scripted decoration, no new game rule | PASS: timeline and pixels in web-render; no `GameState`, `Observation`, collision engine, or synthetic player input |
| IV. Deterministic simulation | PASS: injected seed and ticks, no renderer wall clock | PASS: Rust shuffled deck, fixed event offsets, randomness only at reset/deck boundaries; JS supplies bounded tick deltas |
| V. Multiplayer and bots | PASS: no changes | PASS: decorative cast is independent of occupants and cannot write room state |
| VI. Dual-interface parity | PASS: no gameplay feature | PASS: web-only decoration needs no terminal equivalent; existing rules remain shared |
| Platform, safety, dependencies | PASS: existing Rust/WASM renderer, no new dependency | PASS: additive scalar exports in existing ffi.rs, finite bounds, existing command ABI and CSP; all new assets counted |
| Required merge gates | PASS: retain repository gates | PASS: quickstart retains fmt, clippy, workspace tests/build and scenario checks; not claimed run during planning |

No constitutional exception is required. Specification and plan approval remain prerequisites to implementation.

## Project Structure

### Documentation (this feature)

```text
specs/014-lobby-animation/
  spec.md
  plan.md
  research.md
  data-model.md
  quickstart.md
  checklists/requirements.md
  contracts/lobby-scene.md
```

`tasks.md` is intentionally deferred to `/speckit-tasks`.

### Source Code (repository root)

```text
game/crates/web-render/
  src/lib.rs                  register the new module
  src/lobby.rs                new pure timeline, frame model, command builder
  src/look.rs                 reuse existing side-profile sprite runs
  src/sprites.rs              reuse bomb artwork and effect colours
  src/commands.rs             reuse Command and flatten; no broad refactor
  src/ffi.rs                  additive reset/advance/frame exports
  tests/lobby.rs              new focused timeline and rendering tests
  tests/ffi.rs                extend existing FFI isolation tests
game/crates/web/
  assets/app.js               replace paintLogo with scene lifecycle and renderer wrapper
  assets/index.html          replace logo element; fallback and pause control
  assets/style.css           bounded responsive scene styles
  assets/lobby-scene.png      new first-party static fallback, generated at implementation
  assets/eksplode_re_render.wasm  rebuilt through existing script
  src/assets.rs              embed PNG route and image/png MIME
  js-tests/lobby.test.mjs     extend existing Animator/lifecycle tests
  js-tests/app.test.mjs       replace the old logo assertion, preserve menu actions
  js-tests/glue.test.mjs      extend real wrapper/fake WASM tests
  js-tests/nav-focus.test.mjs extend focus assertions where needed
  tests/http.rs              fallback MIME and full asset-budget coverage
game/README.md               document pause control
game/docs/WEB.md             main-menu scene and accessibility behavior
game/scripts/check-scenarios.sh  register feature acceptance prefixes
```

**Structure Decision**: Keep domain presentation in one focused Rust module and browser orchestration in the existing application module. Reuse existing test files except the new Rust timeline suite. No independent JavaScript animation engine or invented server API.

## Phase 0: Research Decisions

Completed in [research.md](research.md). Source inspection resolved placement, rendering reuse, shuffled-coverage scheduling, lifecycle clocks, graphics failures, asset serving, and test entry points. No unresolved technical decisions block design.

## Phase 1: Design

### Rendering and Choreography

Use a logical 640 by 200 stage, uniformly fitted and clipped to a responsive reserved area. Draw a quiet pixel-art ground and a smaller background baseline, emit background actors first, and mirror directional sprites correctly. Add a small side-view kart body/wheel drawing in `lobby.rs`; the existing top-down gameplay kart is not suitable. Bomb pushing and flattening are authored timed poses, never a collision simulation. Reuse the game's palette and sprite identity; no gradients, card framing, external imagery, or replacement game branding.

Seven five-second definitions are shuffled without replacement: RunDrop, SameLevel, CounterRunBlast, BackgroundRun, KartLeft, KartRight, PushSquash. Each definition has fixed event offsets and guaranteed coverage tags. A deck lasts 35 active seconds; the worst gap between the same required event in successive shuffled decks is at most 65 seconds, leaving margin within FR-007's 90-second target. Suppress identical neighbors at deck boundaries without losing any item. Use injected core RNG; never choose randomness per render frame. See [data-model.md](data-model.md) for states and invariants.

### Browser and WASM Boundary

Add reset(seed), advance(ticks), and frame(width, height, alpha) scalar exports, keeping the shared draw-command output lifetime unchanged. Scene state must not read or overwrite observations, interpolation, looks, picking geometry, or network state. `createRenderer` feature-detects the new exports; older modules get the still fallback, not a fatal error. Paint or copy returned commands before any portrait/game draw overwrites the shared buffer. See [contracts/lobby-scene.md](contracts/lobby-scene.md).

Create the scene controller once during `start()`. Synchronize from actual `screens(state).mainMenu` visibility, modal coverage, document visibility, renderer readiness, live reduced-motion changes, and explicit pause. Do not key activity to a guessed stage name or the room-lobby animator. Retain the timeline when hidden; discard hidden elapsed time and reset only the RAF origin on return. No scene restart on room updates, customization updates, resize, or repeated `updateUi()` calls.

Use a native icon button outside `.menu-actions`, after existing primary actions in focus order; use the project's current icon style, accessible label, tooltip, and `aria-pressed`. The canvas and fallback are decorative and pointer-transparent. Keep the existing h1 and all four menu actions unchanged. Responsive sizing must account for the existing mobile/short-landscape rules rather than extend obsolete `#join-panel > #logo` selectors.

### Fallback and Loading

Ship a first-party static PNG derived from the scene's representative frame. It is present before WASM resolves and remains if exports are missing or scene draw fails. An error in this optional feature must not set the global connection stage to `closed`. Refine `loadGraphics()` so an unavailable module leaves menu navigation and network state intact with the static scene; actual gameplay rendering can still report its graphics error when required. Do not promise playable matches without WASM, hide genuine network failures, or silently accept a broken renderer. If Canvas 2D is unavailable, the PNG still displays.

### Validation and Delivery Boundary

Use [quickstart.md](quickstart.md) for red-first automated coverage, real WASM/browser checks, asset budgets, accessibility and full merge gates. Extend asset accounting to include every embedded page asset, including existing routes omitted by its hard-coded list. Required documentation and the new acceptance-prefix registration remain in implementation scope. Human satisfaction evidence is an explicit later gate, not inferred from screenshots.

Phase 1 ends with this design, research, model, UI/FFI contract, and validation guide. No application code, binary assets, tasks, commits, branches, or running dev server are produced by planning.
