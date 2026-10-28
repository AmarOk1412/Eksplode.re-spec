# Manual and browser checks: web client (feature 002)

Automated checks live in `cargo test` and `node --test`; this file records the checks that need a real
browser. `game/scripts/browser-smoke.mjs` drives a headless Chromium through the page with the
DevTools protocol (join, start, keys, screenshots) and was used for the walkthrough below.

## Walkthrough (quickstart Stage B3), 2026-10-22

Server: `eksplode-re-server --http-port 18080 --game-port 17878 --bots 1 --seed 7`, browser: Chromium
headless 1100x800 through `scripts/browser-smoke.mjs`.

- [X] 1. The page loads at `http://0.0.0.0:18080/` with the name field and Join button (US1 AS1)
- [X] 2. Join: the lobby lists "1 bot (ready)" and "SmokeTest (you, owner)" (US1 AS2)
- [X] 3. Start: the isometric arena appears with a coloured cube per player (red 1, blue 2) and number badges (US1 AS3)
- [X] 4. Keys: ArrowDown, ArrowRight and Space moved the blue cube and dropped a bomb; the HUD shows `bombs 1/1` (US1 AS4)
- [X] 5. Flames of the bot's bomb are drawn as orange diamonds along the row (US1 AS3)
- [X] 6. Browser console is clean (no errors or warnings)
- [ ] 7. Second tab joins as another player; closing it during a match shows "Player N left, a bot took over" (US2 AS3): covered by `crates/web/tests/mixed.rs`, to be confirmed by eye
- [ ] 8. Result announced and the lobby returns (US1 AS5): covered by `websocket.rs::us1_as5_*`, to be confirmed by eye
- [ ] 9. Stopping the server shows "The server stopped (server gone)." and a Retry button (US1 AS6): covered by `js-tests/app.test.mjs`, to be confirmed by eye

## Visual checklist (SC-004, SC-005)

- [X] Four colours are distinguishable (red `#dc322f`, blue `#268bd2`, green `#3cb44b`, yellow `#e6be1e`) and each cube has a legible number badge
- [X] Walls, blocks, flames and the arena floor are distinguishable; cubes are taller than walls so they stay visible behind them (walls 22 px, blocks 18 px, cubes 26 px)
- [ ] Bombs are distinguishable while the player is not standing on them (bomb is a dark pulsing circle)
- [ ] No visible stutter during a chain explosion at 60 fps
- [ ] The arena rescales when the window is resized

## Notes

- A bomb placed under the player's cube is hidden by the cube until the player moves away; this follows the depth order (player layer above bomb layer).
- Items left unchecked need a person with a visible browser window; they are not blocking for the automated gates.

## README quick start on a clean checkout (T079, SC-001)

`git clone game /tmp/eksplode_re-check`, then the README steps literally:

| Step | Result | Time |
|------|--------|------|
| 1. `cargo build --workspace` (dependencies already in the cargo cache) | ok | 36 s |
| 2. `cargo run -p eksplode-re-web -- --bots 1` (other ports to avoid clashes) | banner with page and terminal addresses | a few seconds |
| 3. open the page, Join, Start, move, bomb (`scripts/browser-smoke.mjs`) | HUD rows `Bot` and `You`, console clean | 7 s |
| 4. `join` from a terminal (`scripts/pty-enter-check.py`, owner through the server, 4 Enter variants) | ok | a few seconds |
| 5. `cargo test --workspace` | no failures | 76 s |

From the build to playing in the browser is about 50 s with a warm cargo cache, under the 2 minutes of SC-001. A cold dependency download was not measured. No README gap was found.

## Architecture questions (T079, SC-009)

A fresh review pass answered five questions using only `README.md` and `docs/`:

1. Where do the rules live? In `crates/core` (pure `step`), ARCHITECTURE "Principles" and "core". Correct.
2. How does a key press reach the screen in the browser? `Input` text frame every 50 ms, ws bridge, in-process client, `Host`, `core::step` at 20 Hz, `Snapshot`, `eksplode_re_load` and `eksplode_re_frame`, canvas paint (the sequence diagram). Correct.
3. How do bots plug in? `Bot` trait, `Observation` in, `PlayerInput` out, the host asks a bot for every bot slot each tick; a leaving player's slot gets a bot. Correct.
4. Who can start a match? The lowest slot held by a person (lobby owner); others get `Error{NotOwner}`. Correct.
5. How does a page from another origin get access, and what is the risk? `--allow-origin <ORIGIN>` (or the unsafe `--allow-any-origin`); CORS warning printed; anyone's page from that origin can drive the game (SECURITY, WEB). Correct.

Result: 5 of 5 answered from the documentation alone. The review was done by the author's assistant, not by an independent person, so SC-009 should be confirmed by a human reader.

## Performance and safety results (T082, T083)

- SC-003: key press to the snapshot showing it, p95 about 100 ms with 2 browsers and 2 terminals at 20 Hz (limit 200 ms), `mixed.rs::sc003_*`.
- SC-005: wasm 153 KB (limit 300 KB), whole page 171 KB (limit 500 KB), `http.rs::sc005_*`. Smoothness at 60 fps was not machine-measured; the page uses `requestAnimationFrame` with interpolation, to be confirmed by eye.
- SC-007: 100 of 100 cross-origin upgrades refused by default, `origin.rs::sc007_*`.
- SC-008: Enter starts the match in 20 of 20 attempts for each variant (CR, LF, CRLF, kitty), for the host and for a joiner who is the owner of `eksplode-re-server`.
