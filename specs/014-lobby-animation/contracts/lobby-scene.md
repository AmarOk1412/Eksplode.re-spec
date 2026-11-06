# Contract: Main-Menu Lobby Scene

**Date**: 2026-11-06
**Status**: Proposed; additive design for implementation
**Related**: [spec](../spec.md), [model](../data-model.md), [validation guide](../quickstart.md)

## User Interface Boundary

The feature replaces `canvas#logo` and its old styling/paintLogo wiring inside `section#main-menu`. It does not insert a scene into `#lobby-panel` or alter room controls. Preserve the existing game heading, name input, menu-create, menu-join, menu-customize, menu-options, and retry IDs and behavior.

| Element | Proposed ID | Contract |
|---------|-------------|----------|
| Reserved scene area | lobby-scene | Unframed, bounded, clipped, stable responsive size; logical art ratio 640:200 |
| Canvas | lobby-scene-canvas | aria-hidden, no tabindex, pointer-transparent; backing size follows bounded DPR |
| Still fallback | lobby-scene-fallback | Same-origin PNG, empty alt, aria-hidden; visible before graphics readiness and on reduced motion or failure |
No scene pause/play button or icon is exposed. The existing four-action menu and focus order remain unchanged. Artwork is never a focus target or live announcement. Reduced-motion and visibility preferences control player-facing playback; internal fixture pausing is available only through the test harness.

No animation sound is emitted. All scene content stays inside the reserved region, including explosions and off-stage entrants. Opening Options/Customize or leaving the main menu freezes the scene. The scene must not determine application connectivity or screen navigation.

## HTTP Asset Boundary

Add exactly one static image route: `GET /lobby-scene.png`, returning the first-party representative PNG with `Content-Type: image/png`. Embed through `web/src/assets.rs::lookup()`, use existing cache/security response handling, and retain CSP unchanged. No new endpoint accepts input, no remote resource is needed, and no new JS module route is required.

The PNG and modified compiled WASM count toward the existing 512,000-byte aggregate page limit; WASM also remains within 307,200 bytes. Update tests to count every embedded page route rather than only the old manual subset. A failed decorative image must not remove menu controls or hide the game heading.

## Additive WASM Exports

Proposed names match the current `eksplode_re_*` export convention. Existing frame/portrait/load/looks/picking signatures and the 16-float command stride remain unchanged.

| Export | Arguments | Result and Behavior |
|--------|-----------|---------------------|
| eksplode_re_lobby_reset | seed: u32 | i32 status; 0 initializes/replaces the independent timeline from that seed |
| eksplode_re_lobby_advance | ticks: u32 | i32 status; 0 for 0..15 accepted ticks, 1 for invalid tick count, 2 if not initialized |
| eksplode_re_lobby_frame | width: f32, height: f32, alpha: f32 | usize command count; 0 for uninitialized timeline or invalid input; sampling never advances state |

`width` and `height` must be finite, positive, and at most 8,192. `alpha` must be finite and in [0, 1). Invalid advance requests do not mutate the timeline. Invalid frame requests clear the scene-call output and return zero without changing timeline, observation, looks, or gameplay canvas dimensions. Normal initialized, valid frames must return nonzero command counts.

Reset does not create an Observation or modify saved/participant looks. Tick advancement is deterministic. Frame generation fits the logical stage and emits background before foreground commands. It does not affect the picking geometry from the most recent actual gameplay frame.

Use the existing `eksplode_re_output_ptr()` and `eksplode_re_stride()` for output. A returned slice has `count * 16` f32 values, valid only until a subsequent drawing export or state operation that reallocates output. JS must reconstruct the typed view after the call, paint immediately or copy it, and never cache it across another render call or memory growth. All unsafe memory exchange remains confined to the existing ffi.rs module.

## Browser Renderer Wrapper

Extend the existing `createRenderer()` result with these proposed methods:

| Method | Behavior |
|--------|----------|
| lobbyAvailable() | True only if reset, advance, and frame exports all exist |
| resetLobby(seed) | Validate integer seed 0..4294967295, invoke reset; report unsupported/status failure locally |
| advanceLobby(ticks) | Validate bounded nonnegative integer, invoke advance; report status failure locally |
| lobbyFrame(width, height, alpha) | Return the existing {count, stride, data} command shape after the export |

Missing exports in an older module cause PNG fallback, not an exception escaping `updateUi()` or loss of the menu. A scene-only failure does not invalidate a renderer that can still paint gameplay. A complete module load failure is kept distinct from a network disconnection: leave the menu and networking usable with fallback, report actual graphics limitations through existing error presentation when gameplay needs them, and do not claim gameplay works without its renderer.

## Clock and Randomness Boundary

- Browser obtains one unsigned seed through Web Crypto when available; use a fixed documented seed when unavailable. This is decorative randomness, not a security guarantee. Native tests inject seeds directly.
- Rust owns deck shuffling and authored movement. Browser code cannot randomly choose variants or inspect real player slots to populate the scene.
- Browser converts active monotonic frame deltas into 60 Hz integer ticks plus fractional interpolation. Reject non-finite times, treat backward time as zero, cap a single delta at 250 ms, and discard hidden elapsed time.
- Reset is called once per controller initialization, not on resize, UI updates, or screen return. Hidden screens and reduced motion freeze timeline state; reduced motion displays the still PNG. Internal test-fixture pauses preserve state without exposing a UI control.
- Seven fixed-duration items in every shuffled deck, with fixed cue offsets, guarantee all required actions within a sliding 90-second active interval. Future additions must retain or re-prove that bound.

## Compatibility and Error Contract

No WebSocket or terminal protocol messages, room options, authentication changes, local storage keys, CLI flags, or core rules are introduced. Scene tests must assert that these boundaries remain untouched. Existing library error-handling conventions apply; malformed dimensions and excessive tick requests must not panic. Missing Canvas, missing scene exports, scene draw errors, or blocked WASM must each leave a game-themed still zone and usable menu controls, with no duplicate RAF loop or repeated error dispatch.