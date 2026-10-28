# Quickstart & Validation Guide: Web Client

All commands run from `game/`. Prerequisites: Rust toolchain, a desktop browser, `node` (optional, for the JS tests). Rebuilding the WASM needs `rustup target add wasm32-unknown-unknown` (normal users do not need it: the built module is committed).

## Stage A - Enter fix (US3 AS4, FR-008, SC-008)
```sh
cargo test -p eksplode-re-cli                 # key normalisation, owner start, refusal notices
python3 scripts/pty-enter-check.py          # CR, LF, Ctrl+J, kitty Enter all start; non-owner sees a notice
```
Expected: every Enter variant starts the match in the host lobby; a joined non-owner pressing Enter sees "Only <name> can start the match"; a joined owner starts the match.

## Stage B1 - Render module (FR-003, SC-004, SC-005)
```sh
cargo test -p eksplode-re-web-render
./scripts/build-wasm.sh                     # optional rebuild; prints the size (<= 300 KB)
(cd crates/web/js-tests && node --test)           # loads the .wasm, checks the draw list
```
Expected: 4 distinct coloured cubes with badges, depth order back to front, deterministic buffers, every error code covered.

## Stage B2 - Server (FR-001, 005, 006, 009, 010, 011, 013)
```sh
cargo test -p eksplode-re-web                 # http, origin, websocket, mixed, node tests
cargo run -p eksplode-re-web -- --bots 1      # prints web page and terminal addresses
```
Expected output shows `Web page: http://0.0.0.0:8080/` and `Terminal client: eksplode-re-cli join 0.0.0.0:7878`. Check origin policy:
```sh
curl -i -H 'Origin: http://evil.example' -H 'Connection: Upgrade' -H 'Upgrade: websocket' \
     -H 'Sec-WebSocket-Version: 13' -H 'Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==' \
     http://0.0.0.0:8080/ws          # expect HTTP/1.1 403 (not 101)
cargo run -p eksplode-re-web -- --allow-origin http://localhost:3000   # expect a WARNING line
cargo run -p eksplode-re-web -- --bind 0.0.0.0                         # expect the exposure WARNING
```

## Stage B3 - Browser page (US1, US2)
1. `cargo run -p eksplode-re-web -- --bots 1`, open `http://0.0.0.0:8080/` in a browser.
2. Enter a name, join: the lobby shows your slot marked "you" and the bot.
3. Press Start (or Enter): the isometric arena appears with a coloured cube per player and a number badge.
4. Move with arrows or WASD, bomb with Space or E: bombs pulse, flames spread, blocks break, power-ups appear.
5. Open a second tab: it joins as another player; close it during the match: the other tab shows "Player N left, a bot took over".
6. Wait for the result: winner or draw announced, lobby returns.
7. Stop the server (Ctrl+C): the page shows "server gone" and a Retry button.

Manual visual checklist (SC-004, SC-005): four colours distinguishable, badges legible, walls/blocks/bombs/flames distinguishable, no visible stutter during chain explosions, window resize rescales.

## Stage B3 - Mixed match (US2, SC-002)
```sh
cargo run -p eksplode-re-web -- --bots 1                 # terminal A
cargo run -p eksplode-re-cli -- join 0.0.0.0:7878      # terminal B
# browser tab joins http://0.0.0.0:8080/ ; the first to join (owner) presses Start
```
Expected: all three see the same arena and result; `cargo test -p eksplode-re-web --test mixed` automates this.

## Stage C - CLI options (US3, FR-006, FR-007)
```sh
cargo test -p eksplode-re-cli
cargo run -p eksplode-re-cli -- host --bots 1 --web-port 8081    # terminal host + web page
cargo run -p eksplode-re-cli -- join 0.0.0.0                   # joins on the default port
```
Expected: the host shows the web URL in its lobby; browser and terminal joiners appear in the same lobby; a closed port gives a clear error and exit status 1.

## Stage D - Documentation (US4, FR-015, SC-009)
```sh
cargo test -p eksplode-re-web --test docs       # every crate, flag and message variant is documented
```
Review checklist: README quick start reaches a running browser match in under 2 minutes; ARCHITECTURE answers "where do rules live", "how does a key press reach the screen", "how do bots plug in", "where is the origin check"; SECURITY explains the no-encryption warning and CORS policy.

## Acceptance scenario -> test map
| Scenario | Test location |
|----------|---------------|
| US1 AS1 page loads | `web/tests/http.rs::us1_as1_*`, manual |
| US1 AS2 join/full | `web/tests/websocket.rs::us1_as2_*` |
| US1 AS3 isometric cubes | `web-render/tests/*`, `web/js-tests/render.test.mjs::us1_as3_*`, manual |
| US1 AS4 controls | `web/js-tests/input.test.mjs::us1_as4_*`, `websocket.rs::us1_as4_*` |
| US1 AS5 result and rematch | `websocket.rs::us1_as5_*` |
| US1 AS6 connection lost | `js-tests/app.test.mjs::us1_as6_*`, manual |
| US2 AS1-AS4 | `web/tests/mixed.rs::us2_as*` |
| US3 AS1-AS4 | `web/tests/args.rs`, `cli/tests/keys.rs`, `cli/tests/online.rs::us3_as*`, pty check |
| US4 AS1-AS4 | `web/tests/docs.rs`, review checklist |

## Definition of done
fmt + clippy + tests green (including `node --test` when node exists), wasm builds, docs test green, README quick start verified by following it literally, commits in the `part: short desc` style (`web` and `docs` are new parts; the commit hook is updated).
