# Quickstart: Validate Mouse-Free Game Navigation

## Prerequisites

- Rust stable and Node.js are installed; Chromium is available for the smoke script.
- Work from `game/`. For the physical check, an Xbox-compatible controller must be listed by `jstest-gtk` and the page must be open in a visible, focused browser.

## Automated Checks

```sh
node --test crates/web/js-tests/*.test.mjs
cargo test -p eksplode-re-web --test http
cargo fmt --check && cargo clippy --all-targets -- -D warnings && cargo test --workspace
```

Expected: the navigation tests (rectangle movement, layer stack, controller edges and repeat, adjustable controls, on-screen keyboard) pass, `/nav.js` is served, and no existing test changes behavior.

## Browser Smoke

```sh
cargo run -p eksplode-re-web -- --bots 1 --http-port 8090 --game-port 7890
node scripts/browser-smoke.mjs http://127.0.0.1:8090/ /tmp/nav-smoke
MAIN_FLOW=keyboard node scripts/browser-smoke.mjs http://127.0.0.1:8090/ /tmp/nav-main-keyboard
MAIN_FLOW=controller node scripts/browser-smoke.mjs http://127.0.0.1:8090/ /tmp/nav-main-controller
```

Run each flow against a fresh server process. The default smoke covers settings, OSK entry, lobby controls, peer ownership and a match; the `MAIN_FLOW` variants start from a clean room and cover create, join, lobby, match and the in-match hub using only that input method. Expected: clean console and no movement, bomb or punch input while a layer is open.

For developer controls, start with `cargo run -p eksplode-re-web -- --bots 1 --dev` and run the default smoke. It checks keyboard and controller access to the hub's Developer action, pause toggle and a drop slider.

## Manual Checks, In Delivery Order

1. **Settings**: Press Escape (or Start on the controller). Open Sound, move both sliders with left and right, toggle Mute, press B back to the hub. Open Controls, rebind a key, cancel a capture with Escape, restore defaults with confirmation.
2. **Lobby**: Move through the room settings, change mode and board with left and right, rename the room with the on-screen keyboard, add a bot, open Change look, pick a colour, press Done, mark Ready.
3. **Main screen**: From a fresh load create a room (mode choices, board, time), enter the player name with the on-screen keyboard, join, start, then open the menu hub during the match and check that the character does not move or bomb until it is closed.
4. Unplug the controller with a menu open: keyboard navigation continues and nothing stays held.

If the browser does not list a plugged-in controller, keep the page focused and press a controller button once; see `specs/005-controller-keyboard-controls/quickstart.md` for the diagnosis steps.

**Physical controller check (2026-10-27)**: Not completed in this environment. `jstest-gtk` is installed and was launched, but `/dev/input/js*` is absent and `/proc/bus/input/devices` has no Xbox, gamepad or joystick entry. Browser validation used the synthetic standard-mapped Gamepad harness; repeat the manual checks above when the controller is exposed to Linux.
