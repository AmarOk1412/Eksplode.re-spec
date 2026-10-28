# Quickstart: Validate Mobile Browser Gameplay

## Prerequisites

- Rust stable and Node.js 20 or newer.
- A Chromium-based browser for the browser smoke check.
- For device verification, a current iPhone or Android phone connected to the same network as the game server.

## Automated Checks

Run from `game/`:

```sh
(cd crates/web/js-tests && node --test)
cargo test -p eksplode-re-web --test http
cargo fmt --check
cargo clippy --workspace --all-targets -- -D warnings
cargo test --workspace
```

Expected: touch mapping and lifecycle tests pass; the embedded touch module and match markup are served; existing desktop input, room, navigation and mixed-client tests remain green.

## Browser Smoke

```sh
cargo run -p eksplode-re-web -- --bots 1 --http-port 8090 --game-port 7890
node scripts/browser-smoke.mjs http://127.0.0.1:8090/ /tmp/mobile-gameplay-smoke
```

The mobile smoke flow uses portrait and landscape touch-capable viewports with synthetic touch input. It verifies bottom-left portrait steering, bottom-right actions, Punch hidden without its power, movement in both orientations, simultaneous movement/action, touch cancellation and results persistence. It also checks that no gameplay input is sent while a navigation layer is open.

## Acceptance Scenario Coverage

| Scenario | Verification |
|----------|--------------|
| US1-AS1 | Mobile viewport smoke and physical phone load: the room flow is visible and touch-operable. |
| US1-AS2 | Physical phone walkthrough: enter a name, choose a character, adjust an available setting and reach a match. |
| US1-AS3 | Browser viewport smoke and phone rotation during setup: controls remain in bounds without unintended horizontal page scrolling. |
| US1-AS4 | Documentation review: README and `docs/WEB.md` state mobile browser support and the portrait/landscape control layouts. |
| US1-AS5 | Existing desktop keyboard, mouse and controller browser smoke regression. |
| US2-AS1 | `touch.test.mjs` cardinal-direction tests plus browser smoke while holding and releasing the left zone. |
| US2-AS2 | Browser smoke verifies Bomb; the app test checks Punch visibility follows the player's power, and device check verifies the powered action. |
| US2-AS3 | Pointer-ownership tests: one zone's touch release does not release another zone's active pointer. |
| US2-AS4 | Physical phone check that controls do not scroll, zoom or select the page during match play. |
| US2-AS5 | Unit and browser checks for pointer cancellation, blur and background/foreground neutralization. |
| US2-AS6 | Multi-touch browser smoke: hold movement while pressing an action in the right zone. |
| US2-AS7 | Portrait viewport smoke checks bottom-corner controls and active joystick input. |
| US2-AS8 | Landscape viewport checks preserve the existing horizontal control layout without overlap. |
| US2-AS9 | App/Node test confirms Punch is absent without the power and appears when the local player's snapshot grants it. |
| US2-AS10 | Capability test plus mobile smoke with fine pointer queries, touch-event support and `maxTouchPoints`, representing Firefox Android/hybrid devices. |
| US3-AS1 | App reducer/result-row tests verify winner and all participants' final standings and stats. |
| US3-AS2 | App reducer tests cover Draw, Team and Chaser outcomes; browser smoke checks rendered outcome text. |
| US3-AS3 | Browser smoke delivers the lobby message after `Ended` and verifies the result panel remains visible. |
| US3-AS4 | Browser smoke activates Return to Lobby and verifies the latest lobby appears. |

## Manual Device Checks

1. Open the game on current Firefox for Android, Android Chrome or iOS Safari; complete setup in portrait.
2. Start a match in portrait. Verify the joystick is bottom-left and Bomb is bottom-right; steer in each cardinal direction and confirm release stops movement.
3. Before acquiring Punch, confirm the Punch button is absent. Acquire the Punch power, confirm its button appears, and activate it.
4. Hold movement while pressing Bomb and confirm both work together. Cancel a touch and background/restore the page; verify no action remains held.
5. Rotate to landscape and confirm the existing horizontal layout remains usable. Check safe areas and verify no control overlap, page scroll or zoom occurs during play.
6. End a round in Normal, Team or Chaser mode. Verify the outcome, all participant standings and final stats remain visible after the lobby update.
7. Activate Return to Lobby and verify the latest lobby appears. Repeat the match with desktop keyboard, mouse and controller inputs.

## Validation Record (2026-10-27)

- The web JavaScript suite passed: 169 tests, including Firefox touch-event detection, portrait controls, conditional Punch visibility and result-state behavior.
- The web HTTP suite passed: 20 tests. The web documentation suite passed: 10 tests.
- `cargo fmt --all --check` and workspace clippy passed.
- Both final Chromium CDP flows passed: `MOBILE_FLOW=setup` and `MOBILE_FLOW=controls`. The controls flow emulates Firefox Android with touch events, verifies portrait bottom-corner controls, landscape layout, hidden Punch without power, touch cancellation and result persistence through a lobby message.
- `cargo test --workspace` still fails only in three untouched CLI online tests concerning host warnings and wildcard bind addresses. The feature's web JavaScript, HTTP, docs and browser checks pass.
- T023 is marked complete in `tasks.md`. This automated run used Chromium emulation; per-device observations are not included in this session record.
