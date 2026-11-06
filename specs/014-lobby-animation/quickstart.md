# Quickstart Validation: Animated Lobby Scene

**Date**: 2026-11-06
**Related**: [spec.md](spec.md), [data-model.md](data-model.md), [contracts/lobby-scene.md](contracts/lobby-scene.md)

This guide documents implementation checks and recorded results. Application code, scene exports, tests, and the fallback PNG are now implemented. The five-player review remains a separate human acceptance gate; do not infer it from automated results.

## Prerequisites

- Start at the repository root with its existing `game/` working tree; preserve unrelated changes.
- Install the existing stable Rust toolchain, Node.js, and a browser automation environment; no new application package is required.
- Install the WASM target if absent: `rustup target add wasm32-unknown-unknown`.
- Review the contract for main-menu placement, seven vignette identities, pause semantics, FFI input bounds, and asset limits.

## 1. Establish Red-First Checks

Add failing focused tests for the changed behavior before implementation. Prefix automated acceptance names with `f14_usN_asM_` and register those exact prefixes in `game/scripts/check-scenarios.sh`; keep older regression tests intact except the intentionally replaced old-logo assertion.

From the repository root:

```sh
cargo test --manifest-path game/Cargo.toml -p eksplode-re-web-render f14_
cd game/crates/web/js-tests
node --test --test-name-pattern='f14_' lobby.test.mjs app.test.mjs glue.test.mjs nav-focus.test.mjs
```

Before adding the tests these filters can match zero cases, which is not evidence of success. Verify actual test counts and observe the newly authored tests fail for the missing feature. After implementation rerun the same commands and ensure every added case executes and passes.

## 2. Native and Browser-Glue Validation

From the repository root:

```sh
cargo test --manifest-path game/Cargo.toml -p eksplode-re-web-render
cargo test --manifest-path game/Cargo.toml -p eksplode-re-web --test http
cd game/crates/web/js-tests
node --test
```

Expected coverage: same-seed replay, frame-rate-independent sampling, seven-item deck completeness, nonrepeating boundary neighbors, all required visible cue offsets, worst-case adjacent deck orders, no unfinished objects at rollover, finite commands, invalid input rejection, command-buffer lifetime, no observation/look/picking mutation, no duplicate RAF, no hidden-time catch-up, and live motion changes. Verify new static route MIME and complete aggregate asset accounting.

Use existing fake frame/context/export helpers rather than implement a second animation renderer in tests. Tests should force every authored kind through native fixture timelines; do not add a production query parameter or scene selector merely for test access.

## 3. Build the Actual WASM and Run the Server

From the repository root:

```sh
cd game
sh scripts/build-wasm.sh
cargo test -p eksplode-re-web --test http
```

The build refreshes the embedded `eksplode_re_render.wasm`. The fallback PNG must also exist before compiling the server's new embedded route. Both the 307,200-byte WASM limit and 512,000-byte complete page limit must pass without raising thresholds.

From the repository root, start the real app:

```sh
cargo run --manifest-path game/Cargo.toml -p eksplode-re-web -- --bind 127.0.0.1 --http-port 8080 --game-port 7878
```

Open `http://127.0.0.1:8080/`. If either port is occupied, choose another free HTTP/game port pair and use the matching URL. Assets are embedded, so rebuild and restart after asset changes. Leave a working development server running and report its URL when implementing, not during this documentation-only planning phase.

## 4. Main-Menu Visual and Motion Checks

Use real-browser automation with the built WASM, not only mocked canvas commands. The old browser-smoke script expects an outdated Join-first flow; do not report it as end-to-end coverage without adapting navigation.

1. Load a fresh page. Confirm the former logo zone is replaced, the Eksplode.re heading and all current actions remain, and the canvas paints a visible side-profile runner within one second of the menu becoming usable, including graphics startup latency. Record PNG fallback timing and graphics download latency separately for diagnosis, but do not exclude that latency from SC-001 or count the still PNG as motion.
2. Observe at least 90 seconds of active playback. Record all seven kinds and visible cues: dropping/fuse/explosion, safe exit, opposing and same-direction runners, smaller background runners behind foreground action, both kart directions, lead-character explosion, and bomb push followed by another bomb flattening the character. Verify all sprites face travel and the gag is non-graphic.
3. Use native schedule tests for many fixed seeds and adversarial deck positions; real-browser observation proves actual rendered cues rather than lucky scheduler labels. Check every sliding 90-second window in a longer native trace.
4. Take screenshots at 1440x900, 390x844, 320x568, and 844x390. Also test any smaller width the app already supports. Check variant artwork plus paused and fallback states at each size: no clipping of necessary action, label/control overlap, horizontal overflow, or content shifts.
5. Compare canvas pixels across successive frames to prove nonblank artwork and movement, not only element existence. Capture the push/contact/squash and background-depth moments as well as running/karts. Ensure the game art is visible rather than obscured by a missing asset or opaque fallback.
6. During five minutes of menu use, navigate Create, Join, Customize, Options, and Back with supported input methods; open and close dialogs. Create/join a room and check the room lobby is unchanged. Confirm scene playback stops off-menu and no scene action sends game input or changes readiness, occupants, settings, or statistics.

## 5. Accessibility, Lifecycle, and Failure Checks

1. Tab and controller-navigate the menu; existing default focus and action order remain. Activate the pause icon with keyboard, controller, pointer, and touch as supported. Verify its tooltip, accessible name, and pressed state; decorative canvas/image never enter focus or announcements.
2. Pause during motion; compare successive frames to confirm no movement. Navigate away and back and confirm the explicit pause persists. Resume and verify preserved playback rather than a new random reset.
3. Emulate reduced motion before load and while an explosion is active. The PNG replaces motion immediately. Disable the preference and ensure an explicit pause is not cleared. Test media preference changes without reloading.
4. Hide the tab, open a modal, and navigate away repeatedly. Return after a long inactive interval: no accumulated explosion burst, timeline jump, second loop, or duplicated event handler. Resize repeatedly without changing the current vignette.
5. Serve an older WASM module without the new scene exports, simulate scene-call failure, and simulate unavailable Canvas. In each case the PNG remains and Create/Join/Customize/Options still work. In a separate fresh page, block the WASM response entirely: the main menu and networking must remain usable with fallback; distinguish the expected inability to render a real match from decorative failure. Check genuine network failures still surface normally.
6. Monitor audio and document focus: the feature emits no sound and never moves focus by itself. Verify no full-screen flashes or more than three flashes per second. Inspect the accessibility tree and screenshots while dialogs are active.

## Acceptance Coverage Matrix

Every row requires an automated assertion; visual checks supplement that mapping rather than replace it. Browser actions may reuse existing navigation helpers. Native fixture names identify authored cues without exposing a production testing control.

| Spec Scenario | Automated Verification Surface | Additional Evidence |
|---------------|--------------------------------|---------------------|
| US1 AS1 | app DOM and real-browser startup/first-paint check | Main-menu screenshot and latency |
| US1 AS2 | Rust run/drop pose and direction samples, browser pixel movement | Running and dropped-bomb screenshot |
| US1 AS3 | Rust fuse/blast/cleared samples | Readable fuse and local blast |
| US1 AS4 | Rust rollover and maximum empty-interval assertion | No visible blank transition |
| US1 AS5 | Seeded-order and multi-seed variation tests | Different orders across page visits |
| US2 AS1 | JS focus/input isolation and browser action activation | Pointer, touch, keyboard, controller |
| US2 AS2 | Media-query change and frozen-tick tests; browser reduced-motion emulation | Still scene during former blast |
| US2 AS3 | Pause/resume RAF and button semantics tests | Accessible control and frozen pixels |
| US2 AS4 | Browser bounding-box, overflow and stable-layout assertions | Desktop/mobile/landscape screenshots |
| US2 AS5 | Missing-export/draw/Canvas/load-failure tests plus browser fallback/navigation | Static art despite blocked graphics |
| US2 AS6 | JS scene suspension/reentry and pause-preservation tests | Hidden-tab and screen-return exercise |
| US3 AS1 | Rust SameLevel positions, looks, and directions | Distinguishable foreground runners |
| US3 AS2 | Rust CounterRunBlast direction samples | Opposing runners |
| US3 AS3 | Rust lead-blast, elimination, clear, and next-scene samples | Non-graphic ending |
| US3 AS4 | Rust safe-exit versus elimination coverage | Both outcomes visible |
| US3 AS5 | Rust background baseline/scale/order assertions and browser frames | Clear depth behind foreground |
| US3 AS6 | Rust PushSquash push/contact/squash/clear samples | Second bomb visibly causes flattening |
| US4 AS1 | Rust side-profile rider/kart command samples | Recognizable kart rather than runner |
| US4 AS2 | Rust KartLeft/KartRight travel and facing samples | Both complete crossings |
| US4 AS3 | Rust kart rollover and leftover-object tests | Clean scene transition |

For SC-006, ask five players to identify the depicted actions and find Create, Join, and Options without coaching. At least four must recognize the running/bomb and kart scene and describe the menu as more alive without losing those actions. Record actual results; a deterministic test cannot substitute for this human review.

## 6. Required Repository Gates

From the repository root:

```sh
cargo fmt --manifest-path game/Cargo.toml --all --check
cargo clippy --manifest-path game/Cargo.toml --all-targets -- -D warnings
cargo build --manifest-path game/Cargo.toml --workspace
cargo test --manifest-path game/Cargo.toml --workspace
cd game
bash scripts/check-scenarios.sh
```

Also rerun the complete JS suite and WASM build above. Verify the acceptance-prefix script actually includes feature 014 and checks each scenario. Report any pre-existing unrelated failure without changing unrelated code or claiming that all gates passed. Record automated results, browser evidence, asset sizes, and the separate five-player review before feature completion.

## Implementation Evidence (2026-11-06)

- T001 baseline: actual game branch main, clean game worktree; unrelated outer-workspace changes are preserved. Existing Rust/Node ignore patterns cover build outputs, dependencies, logs, environment files, and editor artifacts; no new ignore file is needed.
- Available stable toolchain: rustc 1.99.1 and cargo 1.99.1; default toolchain is nightly 1.92.0. Use `cargo +stable` explicitly for validation. Node is v22.22.2; wasm32-unknown-unknown is installed.
- Before edits: existing FFI suite passes 14 tests; focused lobby/glue JS suites pass 19 tests. Node emits an existing module-type warning from the ancestor package configuration.
- Pre-implementation asset baseline from research: 435,777 bytes total and 240,384 bytes WASM. Browser smoke uses its existing Chromium/CDP loader, without adding application dependencies.
- T002-T008: native missing-module/export failures and missing JS-method failure observed before implementation; all three foundation Rust tests and nine graphics-glue tests pass afterward.
- T009-T017: missing runner/bomb state, old-logo DOM, and missing browser scene failures observed before implementation. Six native scene tests and 68 existing application/lobby/glue tests pass. Real Chromium startup and moving pixels pass in 139 ms, with a reproducible injected seed; full vignette catalog remains pending. Screenshot: `/tmp/eksplode-re-f14-us1/f14-main-menu.png`.
- Real-browser checks exposed zero-row sprite suppression and f32 interpolation rounding to 1.0; dedicated regressions were observed failing and now pass. Stable WASM target was installed explicitly; rebuilt renderer is 242,461 bytes at this checkpoint.
- T018-T028: all six new JS regressions and missing-PNG HTTP failure were observed red before implementation. They now pass; the PNG security/MIME contract and complete page-size gate pass at 450,777 bytes including 242,461-byte WASM. Chromium passes pause/frozen pixels, live reduced motion, retained pause across all four menu actions, keyboard activation, four viewport sizes, and old-module/scene-error/no-Canvas/blocked-WASM fallback cases without client exceptions. Retained evidence is under `game/docs/screenshots/f14-*.png`; detailed captures are in `/tmp/eksplode-re-f14-us2/`. Physical-controller verification remains explicitly unperformed; automated controller/touch checks are part of final validation.
- T029-T033: all six encounter/gag tests failed before implementation, and real WASM initially produced only background commands for the encounter fixture. All 12 native scene tests now pass, along with FFI isolation and desktop/mobile real-WASM fixture captures. WASM is 249,528 bytes. Inspected background-depth and two-bomb flattening captures are retained under `game/docs/screenshots/`; detailed fixture captures are in `/tmp/eksplode-re-f14-us3/`.
- T034-T038: missing kart bodies and an empty catalog window were observed failing before the final two variants. All 16 scene tests and the complete renderer suite now pass. The native trace checks every sliding window across 16 seeds and 15,000 ticks each, with all 12 visible motifs recurring within 65 seconds. Actual WASM fixture pixels pass for all seven scenes on desktop/mobile, with the kart capture visually inspected. A 90-second real Chromium run advanced 5,435 ticks with exactly one initialization; emulated controller and touch pause/resume passed and no gameplay input was emitted. WASM is 250,775 bytes. Captures: `/tmp/eksplode-re-f14-us4/` and `game/docs/screenshots/f14-kart.png`.

Reproduce the isolated real-browser checks against the running server (from repository root):

```sh
BASE_URL=http://127.0.0.1:8094/ OUT_DIR=/tmp/eksplode-re-f14-check node game/scripts/browser-smoke.mjs --lobby-scene
BASE_URL=http://127.0.0.1:8094/ OUT_DIR=/tmp/eksplode-re-f14-live SCENE_SET=all SCENE_LIVE_SECONDS=300 node game/scripts/browser-smoke.mjs --lobby-scene
```

The second command captures every authored scene with an independent real WASM fixture, then runs the actual page's timeline for five minutes while exercising menus. Its seed injection and counters exist only in the browser harness, not the application. Use `RUSTUP_TOOLCHAIN=stable` for the WASM build/scenario script and `cargo +stable` for the recorded Rust commands on this machine.

## Final Validation Results

- T039-T041: all 20 feature-014 scenario prefixes are checked against compiled Rust test registrations and executed Node assertions; the full scenario checker passes, preserving previous features. All 10 web documentation-contract tests pass. README and WEB describe the delivered main-menu scene and motion controls.
- T042: five-minute actual Chromium playback with repeated Create/Join/Customize/Options navigation passes at 18,076 advanced ticks and one initialization. Emulated standard controllers and touch activate pause/resume. Menu operation emits no gameplay input. A sustained-run modal-suspension regression was observed failing, isolated with a short replay, fixed at the shared navigation-layer boundary, and both the replay and five-minute run now pass.
- Every authored vignette renders at 1440x900, 390x844, 320x568, and 844x390 with no horizontal overflow or overlap with the name input. Paused/reduced-motion and old-module/scene-error/no-Canvas/blocked-WASM states were captured separately. Real room creation still produces the original four-card bot-filled room lobby and stops scene ticks. Detailed final captures are in `/tmp/eksplode-re-f14-final-matrix/` and `/tmp/eksplode-re-f14-final-live/`.
- T043: stable workspace fmt (`--all --check`), workspace/all-targets Clippy with `-D warnings`, workspace build, WASM build, full scenario checker, full HTTP/documentation tests, and complete test suites pass. Workspace tests: 912 passed, zero failures across 78 completed targets. JavaScript tests: 229 passed, zero failures. Logs: `/tmp/eksplode-re-f14-workspace-tests.log` and `/tmp/eksplode-re-f14-js-tests.log`.
- Final embedded asset gate: 250,775-byte WASM below 307,200 bytes; 459,091-byte complete page including PNG and version below 512,000 bytes. No limit was increased and no static route was omitted from accounting. Editor diagnostics and whitespace checks are clear.
- Development URL: `http://127.0.0.1:8094/`; terminal-client port is 7894. The game server remains running for review. No branch, commit, dependency, protocol, room setting, or gameplay-rule change was made; unrelated outer-workspace edits and reviewer-owned checklists were left intact.

## Remaining Human Review

**Later user revision (2026-11-06)**: The scene pause/play icon and its DOM/CSS/event wiring were removed. Reduced-motion and automatic suspension remain. Current focus tests require no scene control, and browser fixtures pause through test-only controller capture rather than a UI button. Earlier pause-button and input results above describe the prior implementation, not a current user-facing control.

T044 is intentionally unchecked. Five real players have not provided the SC-006 recognition/usability review. Do not claim the feature fully accepted until at least four of five identify the running/bomb and kart activity, find Create/Join/Options without coaching, and describe the menu as more alive without losing those actions. Record their actual results here when available.

Input verification used Chromium's real page with emulated standard gamepads and touch; no physical controller or additional browser engine was available for hardware/cross-browser certification. This does not replace the separate five-player review.