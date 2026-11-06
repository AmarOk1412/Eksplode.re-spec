# Research: Animated Lobby Scene

**Date**: 2026-11-06
**Scope**: Phase 0 decisions for [spec.md](spec.md). Read-only research examined the current working tree, including its existing uncommitted changes. Decisions below describe planned changes, not implemented symbols.

## 1. Target and Scope

**Decision**: Replace only the main-menu logo zone. Leave room-lobby portraits, actions, and layout unchanged. Preserve all existing main-menu actions, including Customize.

**Rationale**: `game/crates/web/assets/index.html` places `canvas#logo` in `section#main-menu`; `#lobby-panel` has no logo. The user explicitly selected "Replace the main-menu logo" when asked. `app.js::paintLogo()` renders the static personalized character and is called by both `updateUi()` and `renderEditor()`. The spec's initial room-lobby assumption has been corrected before design.

**Alternatives considered**: Adding a new room-lobby scene or animating both screens would expand the confirmed scope. Renaming or removing existing menu actions would be unrelated cleanup.

## 2. Renderer Ownership and Artwork

**Decision**: Add a pure decorative timeline and command builder to Rust `web-render::lobby`; reuse existing pixels and emit the existing command format. JS only schedules frames and paints.

**Rationale**: The constitution requires graphics rendering in web-render WASM. `look.rs::runs_view(look, view, frame, rows)` supports genuine Left/Right profiles and Idle/StepA/StepB frames. `sprites.rs` exposes BOMB, bomb_colour, and flame/smoke colours. `commands.rs` exposes Command, flatten, and a 16-float stride. The existing private Pen and gameplay command builder depend on observations, so they are not an appropriate public abstraction for this feature.

**Alternatives considered**: JS Canvas choreography duplicates rendering ownership and violates the project constraint. CSS-only silhouettes would lose the established character identity. A new animation package adds weight without solving a missing domain requirement. Reusing the gameplay kart directly is unsuitable because its art is top-down and its placement depends on core track rules; add a small authored side-view kart instead.

## 3. Randomness With Bounded Variety

**Decision**: Use seven fixed five-second vignettes and a Fisher-Yates deck driven by injected `eksplode_re_core::Rng`. Shuffle each deck and avoid identical neighbors at its boundary. Events have fixed offsets within their vignette. Randomness is consumed only when creating/refilling a deck.

**Rationale**: `core/src/rng.rs` already provides new(u64), next_below(u32), and next_unit(). No new RNG dependency is needed. Independent random picks can omit a requested action indefinitely and cannot satisfy FR-007. Every deck includes RunDrop, SameLevel, CounterRunBlast, BackgroundRun, KartLeft, KartRight, and PushSquash; durations and authored tags guarantee coverage.

For seven five-second items, one deck lasts 35 seconds. If an event's item is first in one deck and last in the next, its occurrence gap is at most `(2 * 7 - 1) * 5 = 65` seconds. Fixed event offsets cancel in that bound. The first occurrence is within 35 seconds from reset. Thus every sliding 90-second active-playback window includes all motifs, even across arbitrary deck permutations. This bound depends on keeping all event offsets and durations fixed; any future variable timing must re-prove the bound. Tests must exercise adversarial adjacent deck orders, not merely sample lucky seeds.

**Alternatives considered**: A fixed playlist is predictable and contradicts random selection. Pure random selection cannot guarantee variety. A weighted history scheduler is more complex than the small deck needs.

## 4. Decorative State, Not Simulation

**Decision**: Author travel, bomb fuse, blasts, pushes, squash, smoke, and clearing as deterministic tick-based visual cues. Use fictional fixed looks and no real player identities.

**Rationale**: The new request is an ambient cartoon sequence. No collision resolution, damage, fuse rule, match statistics, or kart mechanics belong in frontend decoration. The spec assumes an incoming second bomb performs the flattening gag. State cannot touch `Observation`, `Interp`, room reducers, actual inputs, or `setLooks`. Draw background characters before foreground ones: `paintCommands()` uses array order, not Command.layer, for painter ordering.

**Alternatives considered**: Creating a fake match would couple decorative scenes to rule changes and enlarge test scope. Reading live occupants would make the scene inconsistent in empty rooms and could imply real events.

## 5. WASM Interface and Lifetime

**Decision**: Add scalar reset/advance/frame exports and renderer wrapper methods; reuse the existing output pointer and stride. Keep an optional lobby timeline separate from gameplay state in the existing FFI owner. All pointer work stays in ffi.rs.

**Rationale**: `ffi.rs` uses a thread-local RefCell<State> with obs, interp, out, canvas, and looks. `createRenderer()` constructs Float32Array views after render calls. Shared output is overwritten by portraits or match frames, so it must be consumed immediately and views reconstructed after calls that may grow memory. New scene drawing must not update the stored game canvas or picking result.

**Alternatives considered**: A JSON frame protocol incurs allocations and another schema without need. A second WASM module or canvas library duplicates loading. Reusing participant slots for decorative looks risks corrupting customization and portraits.

## 6. Lifecycle and Accessibility

**Decision**: Use one controller owned by start(), borrowing the idempotent scheduling pattern of Animator without changing the existing room-lobby animator. Feed fixed 60 Hz ticks from active RAF deltas, bounded to 250 ms per callback. Freeze elapsed state during explicit pause, hidden screens/tabs, and modal coverage. Reduced motion uses a representative still scene; listen for preference changes immediately.

**Rationale**: Current `Animator::setActive()` prevents duplicate loops but resets its time origin whenever reactivated. Current `reducedMotion()` samples matchMedia without a change subscription. `screens(state).mainMenu`, not stage === lobby, controls the target's visibility. `nav.js` includes native buttons automatically and restores focus by stable IDs; pause belongs outside `.menu-actions` so the action contract and default focus are preserved.

**Alternatives considered**: A perpetual RAF wastes work while hidden. Catching up a long delta can burst explosions and violate reduced-motion expectations. Adding scene actors to navigation or accessibility trees adds meaningless focus stops. Persisting pause in local storage is outside the spec.

## 7. Failures, CSP, and Asset Budget

**Decision**: Keep a same-origin representative PNG beneath the optional canvas; embed it explicitly with image/png MIME. Feature-detect exports and isolate failures. Include the fallback and all existing embedded assets in the HTTP budget check.

**Rationale**: `web/src/assets.rs::lookup()` enumerates routes via include_bytes!; arbitrary new files are not served. CSP only allows same-origin script/style/image resources, with no external library required. Existing `loadGraphics()` moves the global state to closed on a load error, which conflicts with FR-014 for a usable main menu; narrow error handling must distinguish graphics readiness from connectivity. This does not make matches playable without graphics. The WASM build script caps 307,200 bytes; `web/tests/http.rs` caps a manually enumerated page total at 512,000 bytes. Do not increase limits or omit assets to make tests pass.

**Measured baseline**: Summing every current include_bytes! page asset gives 435,777 bytes; the WASM is 240,384 bytes. Remaining headroom is 76,223 bytes overall and 66,816 bytes for WASM. The new implementation must be remeasured after adding the fallback and rebuilding. First-motion checks must include graphics loading in the one-second usable-menu target; showing the initial still PNG does not itself satisfy motion readiness.

**Alternatives considered**: A WASM-only still frame cannot survive missing WASM. A blank rectangle does not satisfy the game-themed fallback. Replacing the complete graphics-loading architecture would exceed scope.

## 8. Verification Surfaces

**Decision**: Reuse Rust FFI tests and existing JS lobby, application, glue, and navigation tests; add one focused Rust timeline/render suite. Use browser automation on the actual compiled module for pixels, motion, all variants, responsive screenshots, and control navigation.

**Rationale**: `app.test.mjs` currently asserts the presence of the old personalized `#logo`; update that intentional assertion while retaining the four-action menu contract. `lobby.test.mjs` offers fakeFrames and lifecycle coverage. `glue.test.mjs` offers fake command contexts. HTTP tests already check embedded WASM and content types. `game/scripts/browser-smoke.mjs` has an outdated Join-first flow, so it cannot be used unchanged as acceptance evidence; navigate the main menu explicitly in browser automation instead. Existing `check-scenarios.sh` does not register feature 014; implementation must add its actual scenario prefixes.

**Alternatives considered**: Node fakes alone cannot prove visual assets render. Screenshots alone cannot prove deterministic scheduling, state isolation, or user satisfaction. Broad changes to unrelated gameplay tests are unnecessary.

## Resolution Status

All technical unknowns are resolved for Phase 1. The user-confirmed placement is in the spec; the incoming-bomb interpretation, silent playback, decorative identities, and session-only pause are documented assumptions. Timing, performance, size, and player-review outcomes remain implementation acceptance gates, not results measured during planning.