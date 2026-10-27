---

description: "Task list for the Web Client and Project Documentation (feature 002)"
---

# Tasks: Web Client and Project Documentation

**Input**: Design documents from `/specs/002-web-client/`

**Prerequisites**: plan.md, spec.md, research.md, data-model.md, contracts/ (http-and-websocket, wasm-render-api, cli-and-server-options), quickstart.md

**Tests**: REQUIRED (constitution principle II, FR-016). In every phase the test tasks come first and MUST be seen failing before the matching implementation tasks.

**Organization**: Grouped by user story. The shared protocol and Enter-fix work that FR-017 puts first lives in Phase 2 (Foundational) because every story depends on it.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: can run in parallel (different files, no dependency on incomplete tasks)
- **[Story]**: US1..US4 (user story phases only)
- All paths are relative to `game/` (the game repository) unless prefixed with `specs/`
- Test names for spec scenarios follow `usX_asY_<what>` (X = story, Y = acceptance scenario number)

## Commit Convention (applies to every task group)

Commit in `game/` when a task or small group of tasks is green, using:

```text
part: short desc (subject <= 80 characters)

long description to explain the feature
on multiple lines
```

`part` is one of `core`, `bots`, `net`, `cli`, `web`, `render`, `build`, `docs`, `test` (`web` and `render` are new; T004 updates the hook). Each `Commit:` note below is a suggested subject.

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: housekeeping, governance and empty crates

- [X] T001 In `specs/001-bomberman-multiplayer/tasks.md` mark T070-T087 as closed with the note `replaced by specs/002-web-client` (change `- [ ]` to `- [X]` and append ` (replaced by 002)`), because the native GUI is dropped (research R12)
- [X] T002 Revert the unfinished GUI-preparation edit in `crates/cli/src/app.rs` with `git -C game checkout -- crates/cli/src/app.rs` (it only added `tick_with`, `is_human`, `tick_hz`, `toggle_pause`) and confirm `cargo build` passes
- [X] T003 Amend `.specify/memory/constitution.md` to version 1.1.0 (MINOR): in "Technology & Platform Constraints" replace the crate list `core`, `bots`, `cli`, `gui` with `core`, `bots`, `net`, `cli`, `web`, `web-render`; state that the graphical version is a web page whose rendering runs as WebAssembly; allow `unsafe` only in `crates/web-render/src/ffi.rs` (WASM memory exchange, with tests); allow presentation-only HTML/JS under `crates/web/assets/`; add the Sync Impact Report and set `Last Amended` to 2026-10-22
- [X] T004 [P] Update `.githooks/commit-msg` and `.githooks/test-commit-msg.sh` to accept the parts `web` and `render` (pattern `^(core|bots|net|cli|web|render|gui|build|docs|test): .+`) with new test cases, and update the Commit Convention in `README.md`
- [X] T005 [P] Create crate `crates/web-render/` (`Cargo.toml`: name `eksplode-re-web-render`, `crate-type = ["cdylib", "rlib"]`, deps `eksplode-re-core` with feature `serde`, `serde_json`; `[lints]` workspace but `ffi.rs` will opt out) with an empty `src/lib.rs`, and add `[profile.wasm-release]` (inherits `release`, `opt-level = "z"`, `lto = true`, `panic = "abort"`, `strip = true`, `codegen-units = 1`) to `Cargo.toml`
- [X] T006 [P] Create crate `crates/web/` (`Cargo.toml`: name `eksplode-re-web`, lib `eksplode_re_web`, bin `eksplode-re-server`; deps `eksplode-re-core` (serde), `eksplode-re-net`, `eksplode-re-bots`, `serde_json`, `tungstenite`; dev-deps `tungstenite`) with `src/lib.rs`, `src/main.rs` printing nothing yet, and empty `assets/` and `js-tests/` directories
- [X] T007 Add CI steps to `.github/workflows/ci.yml`: `rustup target add wasm32-unknown-unknown`, `./scripts/build-wasm.sh`, `node --test crates/web/js-tests` (skip if node missing), and `python3 scripts/pty-enter-check.py`
- [X] T008 Run `cargo fmt --check`, `cargo clippy --workspace --all-targets -- -D warnings`, `cargo test --workspace`. Commit: `build: add web crates, commit parts and CI steps for the web client`

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: owner-based lobby commands and the Enter fix (plan stage A: FR-008, SC-008, US3 AS4). Everything else uses them.

**CRITICAL**: no user story work before this phase is complete

### Tests first (must fail)

- [X] T009 [P] Lobby owner tests in `crates/core/tests/lobby.rs`: `owner()` is the lowest slot holding `Local` or `Remote`, `None` when only bots or empty, moves to the next human when the owner slot is cleared; `last_bot()` returns the highest `Bot` slot or `None`
- [X] T010 [P] Protocol codec tests in `crates/net/tests/codec.rs`: `ClientMsg::{Start, AddBot, RemoveBot}` round-trip as `{"type":"Start"}`, `{"type":"AddBot"}`, `{"type":"RemoveBot"}`; `Error` codes `NotOwner`, `CannotStart`, `WrongPhase`, `NoSlot` round-trip
- [X] T011 [P] Host command tests in `crates/net/tests/lobby.rs`: owner `Start` starts the match; non-owner `Start` gets `Error{NotOwner}`; `Start` while a match runs gets `Error{CannotStart}`; `AddBot` by owner fills the first free slot and broadcasts `Lobby`; `AddBot` with 4 slots taken gets `Error{NoSlot}`; `RemoveBot` frees the highest bot slot; `AddBot`/`RemoveBot` during a match get `Error{WrongPhase}`; ownership moves when the owner leaves; `Host::connect(false)` returns a client that has not joined and can send `Join` itself
- [X] T012 [P] Key normalisation tests in `crates/cli/tests/keys.rs`: `normalize` maps `Enter`, `Char('\r')`, `Char('\n')`, `Ctrl+J`, `Ctrl+M` to `KeyCode::Enter`; plain `j` stays `j`; `Ctrl+C` and other Ctrl/Alt combinations are ignored (return `None`); `Release` events are ignored
- [X] T013 [P] Online session tests in `crates/cli/tests/online.rs`: `us3_as4_enter_starts_for_the_owner_client` (a joined first client presses Enter and the match starts), `us3_as4_enter_by_non_owner_shows_a_notice` (notice contains `Only` and the owner's name), `us3_as4_lf_and_ctrl_j_start_the_match`, `a_refused_command_is_shown` (server `Error` text becomes the notice), `b_and_x_work_for_a_joined_owner`
- [X] T014 [P] Create `scripts/pty-enter-check.py` (pseudo-terminal harness from the plan: spawns `target/debug/eksplode-re-cli host`, sends CR, LF, CRLF and `ESC[13u` and asserts the game screen appears for each; spawns host plus `join` and asserts a non-owner Enter shows `Only`), plus `crates/cli/tests/pty.rs` that runs it when `python3` exists (skips with a printed notice otherwise)

### Implementation

- [X] T015 [P] Implement `Lobby::owner()` and `Lobby::last_bot()` in `crates/core/src/lobby.rs` (T009)
- [X] T016 [P] Add `ClientMsg::{Start, AddBot, RemoveBot}` to `crates/net/src/protocol.rs` with doc comments (additive; `PROTOCOL_VERSION` stays `1`) (T010)
- [X] T017 Implement owner-checked handling in `crates/net/src/host.rs`: `Start` (error `CannotStart` if a match runs or no participant), `AddBot` (`NoSlot` when full), `RemoveBot` (uses `Lobby::last_bot`), `WrongPhase` during a match, `NotOwner` for non-owners; every refusal sends `ServerMsg::error(code, message)` naming the owner (`only <name> can start the match`) and counts toward the 10-error limit; keep `Host::start_match/add_bot/clear_slot` for the in-process host (T011)
- [X] T018 Add `Host::connect(local: bool) -> Result<Client, NetError>` to `crates/net/src/host.rs` returning an in-process `Client` that has not joined, and re-implement `local_client` as `connect(true)` plus `Join` (T011)
- [X] T019 Add `crates/cli/src/input.rs::normalize(KeyEvent) -> Option<KeyCode>` per T012 and use it for every key event in `crates/cli/src/app.rs` (all three loops), so modifiers are no longer dropped
- [X] T020 Rewrite `Online::lobby_action` in `crates/cli/src/online.rs` to send `ClientMsg::Start`, `AddBot`, `RemoveBot`, `SetReady` for joined clients and to call the in-process host only for the hosting process; show refusals from `ServerMsg::Error` (code `NotOwner`, `CannotStart`, `WrongPhase`, `NoSlot`) as the notice; update the lobby footer text in `render_lobby` to show `Enter start (owner)  b add bot  x remove bot  r ready` for everyone
- [X] T021 Run T009-T014 until green, run `python3 scripts/pty-enter-check.py`, run all gates. Commit: `core: add lobby owner`, `net: add owner commands and in-process connect`, `cli: normalise Enter keys, let the owner start and show refusals`

**Checkpoint**: every Enter variant starts a match for the owner on any client; refusals are visible; the protocol is ready for the web bridge

---

## Phase 3: User Story 1 - Join a running match from a web browser (Priority: P1) MVP

**Goal**: a server that serves the page and WebAssembly graphics, and accepts browser players over WebSocket (FR-001 to FR-004, FR-009, FR-010, FR-012, FR-013, plan stages B1-B3)

**Independent Test**: `cargo run -p eksplode-re-web -- --bots 1`, open the printed address in a browser, join, start, and play a round to its result (quickstart Stage B3)

### Tests for User Story 1 (write first, must fail)

#### Render module (B1)

- [X] T022 [P] [US1] Projection tests in `crates/web-render/tests/iso.rs`: `project(x, y, z)` gives `sx = (x - y) * 32`, `sy = (x + y) * 16 - z`; `unproject` inverts it on the ground plane; `fit(width, height, canvas)` centres the arena with a 16 px margin and scales to fit for canvases of 400x300, 800x600 and 1920x1080
- [X] T023 [P] [US1] Depth tests in `crates/web-render/tests/depth.rs`: `us1_as3_` objects are sorted by `(x + y, layer)`; a player behind a block is drawn before it; floors come first; equal keys keep a stable order
- [X] T024 [P] [US1] Cube and palette tests in `crates/web-render/tests/cube.rs`: faces are left 0.8, right 0.6, top 1.0 brightness of the player colour; palette is red `#dc322f`, blue `#268bd2`, green `#3cb44b`, yellow `#e6be1e` with pairwise RGB distance > 100; the badge text is the slot number 1-4
- [X] T025 [P] [US1] Interpolation tests in `crates/web-render/tests/interp.rs`: position lerps between the previous and current tick, `alpha` is clamped to `0.0..=1.0`, equals the target at 1.0, and loading a new tick moves current to previous
- [X] T026 [P] [US1] Command buffer tests in `crates/web-render/tests/commands.rs`: for a sample 4-player observation fixture (`crates/web-render/tests/fixtures/observation.json`, generated from `GameState::new(Config::default(), 7, [true; 4])`) the buffer has stride 16, exactly 4 player cubes (3 quads + 1 badge text each) with distinct colours, one quad per wall box face set, eliminated players absent, bombs as circles whose radius pulses with `ticks_left`, flames translucent (`a < 1`), power-ups with letters `B`/`R`/`S`; two `frame` calls with equal inputs give equal buffers
- [X] T027 [P] [US1] FFI tests in `crates/web-render/tests/ffi.rs` (the only test file allowed `unsafe`): `eksplode_re_alloc(0)` and `eksplode_re_alloc(2_000_000)` return 0; alloc/free round trip; `eksplode_re_load` returns `0` for the fixture, `1` for invalid UTF-8, `2` for `{}` and for random bytes (fuzz loop of 1000 inputs never panics), `3` for an arena wider than 64; `eksplode_re_frame` returns 0 before any load; `eksplode_re_stride()` is 16

#### Server (B2)

- [X] T028 [P] [US1] HTTP tests in `crates/web/tests/http.rs`: `us1_as1_` `GET /`, `/index.html`, `/app.js`, `/style.css`, `/eksplode_re_render.wasm` return 200 with the exact content types and the headers `Cache-Control`, `X-Content-Type-Options`, `Referrer-Policy`, `Content-Security-Policy` from the contract; `HEAD` returns headers only; `/nope` is 404; `POST /` is 405 with `Allow: GET, HEAD`; a request head over 8 KiB is 431; a silent connection is closed after 5 s (use a shortened timeout option); the 65th concurrent connection gets 503 (`max_connections` 64)
- [X] T029 [P] [US1] Origin tests in `crates/web/tests/origin.rs`: same-origin upgrade (Origin equals `http://<Host header>`) gets 101; `Origin: http://evil.example` gets 403 and nothing reaches the match; missing `Origin` from loopback is accepted; with `AllowedOrigins::List` an exact match is accepted and a different one refused; with `Any` everything is accepted; relaxed modes add `Access-Control-Allow-Origin` to static responses; `warnings(&ServerOptions)` returns the exact WARNING lines for relaxed origins and for a non-loopback bind and none for the defaults
- [X] T030 [P] [US1] WebSocket tests in `crates/web/tests/websocket.rs` using a `tungstenite` client: `us1_as2_join_assigns_a_slot_and_a_fifth_join_is_refused`, `us1_as4_input_moves_the_player` (send `Input` frames, see `Moved` in a `Snapshot`), `us1_as5_round_result_and_return_to_lobby` (short `round_limit_ticks`), `owner_start_over_websocket`, `non_owner_start_gets_notowner`, binary frames get `Error{Malformed}`, text frames over 64 KiB close the socket, ten malformed frames close it (close code 1008), invalid names get `Rejected{BadName}`
- [X] T031 [P] [US1] Options tests in `crates/web/tests/args.rs`: defaults (`bind 127.0.0.1`, `http_port 8080`, `game_port 7878`, `bots 0`, no seed, `SameOrigin`, `max_connections 64`); `--http-port` and `--game-port` accept `1-65535` (and `0` only through the library API for tests); `--bots` accepts `0-3`; `--bind` must be a valid IP; `--allow-origin` is repeatable and rejects values without a scheme; `--max-connections` must be `>= 4`; unknown option and missing value are usage errors (exit code 2); `--help` and `--version`
- [X] T032 [P] [US1] Server lifecycle tests in `crates/web/tests/lifecycle.rs`: starting on a busy `--http-port` returns an error naming the port; `Server::shutdown()` sends `Error{HostLeft}` to every WebSocket and TCP client and closes them; the startup banner contains `Web page: http://`, `Terminal client: eksplode-re-cli join` and `Stop:`

#### Page (B3)

- [X] T033 [P] [US1] WASM-in-Node tests in `crates/web/js-tests/render.test.mjs` (node:test, no dependencies): load `assets/eksplode_re_render.wasm`, push the fixture observation through `eksplode_re_alloc`/`eksplode_re_load`, assert `us1_as3_` four distinct cube colours, back-to-front order, deterministic buffer, each load error code, and `eksplode_re_stride() === 16`
- [X] T034 [P] [US1] Input helper tests in `crates/web/js-tests/input.test.mjs`: `us1_as4_` key state to input (arrows + Space and WASD + E, last pressed direction wins, bomb is one-shot, releasing the last key stops movement, `blur` clears all keys)
- [X] T035 [P] [US1] App logic tests in `crates/web/js-tests/app.test.mjs`: `us1_as6_` page state machine (`Connecting`, `Joining`, `Lobby`, `Playing`, `Closed`) driven by fake `ServerMsg`s; `Rejected{Full}`, `BadName`, `InMatch`, `BadVersion` map to plain-language notices; socket close shows "server gone" with Retry; Retry restarts at `Connecting`; hidden tab sends `Ping` every 2 s
- [X] T036 [P] [US1] Node bridge test in `crates/web/tests/node.rs` that runs `node --test crates/web/js-tests` when `node` is on the PATH and prints a skip notice otherwise

### Implementation for User Story 1

#### Render module (B1)

- [X] T037 [P] [US1] Implement projection, `fit` and `unproject` in `crates/web-render/src/iso.rs` (T022)
- [X] T038 [P] [US1] Implement cube geometry, shading and the palette in `crates/web-render/src/cube.rs` (T024)
- [X] T039 [P] [US1] Implement interpolation state in `crates/web-render/src/interp.rs` (T025)
- [X] T040 [US1] Implement the draw list and command buffer (stride 16, kinds 0-3, ordering by `(x + y, layer)`, floors first, wall and block boxes, flames, power-ups, pulsing bombs, cubes with badges) in `crates/web-render/src/depth.rs` and `crates/web-render/src/commands.rs` (T023, T026; depends on T037-T039)
- [X] T041 [US1] Implement the FFI in `crates/web-render/src/ffi.rs` with `#![allow(unsafe_code)]` limited to this module and a comment explaining why: `eksplode_re_alloc` (0 for 0 or > 1 MiB), `eksplode_re_free`, `eksplode_re_load` (codes 0-3, arena <= 64x64), `eksplode_re_frame`, `eksplode_re_output_ptr`, `eksplode_re_stride`, `eksplode_re_pick_tile`; state lives in a `thread_local!` `RefCell` so no `static mut` (T027)
- [X] T042 [US1] Create `scripts/build-wasm.sh` (`set -eu`; builds `-p eksplode-re-web-render --target wasm32-unknown-unknown --profile wasm-release`, copies `target/wasm32-unknown-unknown/wasm-release/eksplode_re_web_render.wasm` to `crates/web/assets/eksplode_re_render.wasm`, prints the size and fails above 300 KB), run `rustup target add wasm32-unknown-unknown` once, run it and commit the artifact (T033)

#### Server (B2)

- [X] T043 [P] [US1] Implement `ServerOptions`, `AllowedOrigins` and argument parsing (`--bind`, `--http-port`, `--game-port`, `--bots`, `--seed`, `--allow-origin`, `--allow-any-origin`, `--max-connections`, `-h/--help`, `-V/--version`, usage error exit code 2) in `crates/web/src/args.rs` with the ranges `ports 1-65535`, `bots 0-3`, `max_connections >= 4` quoted from data-model.md (T031)
- [X] T044 [P] [US1] Implement the origin policy and `warnings(&ServerOptions) -> Vec<String>` in `crates/web/src/origin.rs` (T029); warning texts exactly as in contracts/cli-and-server-options.md, each starting with `WARNING:`
- [X] T045 [P] [US1] Embed assets in `crates/web/src/assets.rs` (`include_str!`/`include_bytes!` of the five files, content-type table, common headers incl. the exact CSP from the contract)
- [X] T046 [US1] Implement the HTTP layer in `crates/web/src/http.rs`: accept loop with `max_connections` accounting (503 beyond), 5 s read timeout, request-head limit 8 KiB (431), `GET`/`HEAD` only (405 with `Allow`), the fixed route table, peek-based detection of `Upgrade: websocket` on `/ws`, `Access-Control-Allow-Origin` in relaxed modes (T028; depends on T044, T045)
- [X] T047 [US1] Implement the WebSocket bridge in `crates/web/src/ws.rs` and `crates/web/src/bridge.rs`: `tungstenite::accept_hdr` with the origin check (403 on refusal), message limit 64 KiB, text frames decoded with `eksplode_re_net::codec::decode::<ClientMsg>` and forwarded through `Host::connect(false)`, `ServerMsg` serialised to text frames, binary frames answered with `Error{Malformed}`, error counter closing at ten (close code 1008), 20 ms read timeout loop pumping both directions, clean close on host shutdown (1001) (T030; depends on T018, T046)
- [X] T048 [US1] Implement `Server::start(options)` and `Server::shutdown()` in `crates/web/src/lib.rs` (starts `net::Host` on the game port with `bots` pre-added, then the HTTP server; busy port error names the port; `shutdown` sends `HostLeft`) and the banner/addresses in `crates/web/src/addr.rs` (loopback plus the primary LAN address via the UDP-connect trick, no packet sent) (T032)
- [X] T049 [US1] Implement `crates/web/src/main.rs`: parse args, print warnings to stderr and the banner to stdout, stop cleanly on Ctrl+C (std has no signal handling, so add the small `ctrlc` crate to `crates/web/Cargo.toml` and justify it in the commit body: it sets a handler that sends on an `mpsc` channel, the main thread then calls `Server::shutdown()` and exits 0), exit codes 0 (clean stop), 1 (runtime error, for example port in use), 2 (usage error) (T032)

#### Page (B3)

- [X] T050 [P] [US1] Create `crates/web/assets/index.html` (canvas, lobby panel with name field, Join, Start, Add bot, Remove bot, Ready buttons, HUD list, notice line, Retry button; `<script type="module" src="/app.js">`; no inline scripts or styles because of the CSP) and `crates/web/assets/style.css` (dark theme, responsive canvas, player colour chips)
- [X] T051 [US1] Create `crates/web/assets/app.js` pure helpers (exported for node tests): key state tracker producing `Input` (T034), message reducer implementing the page state machine and notices (T035), WebSocket client with `Join`, 20 Hz input timer (50 ms), `Ping` every 2 s while `document.hidden`, `blur`/`visibilitychange` clearing keys, Retry
- [X] T052 [US1] Extend `crates/web/assets/app.js` with the WASM glue: `instantiateStreaming`, stride assertion, copy each `Snapshot.state` into WASM memory with `eksplode_re_alloc`/`eksplode_re_load`, `requestAnimationFrame` loop calling `eksplode_re_frame(canvas.width, canvas.height, alpha)` with `alpha = (now - lastSnapshot) / 50`, painting commands from a `Float32Array` view (quad, circle, text, triangle), resizing the canvas with `devicePixelRatio`, DOM HUD (per-player bombs/range/speed, own slot marked "you"), results and rematch via the lobby, clear error screen when WebAssembly or WebSocket is unavailable (T033, T035)
- [X] T053 [US1] Run T022-T036, rebuild the wasm, run `cargo run -p eksplode-re-web -- --bots 1`, open the page in a browser and walk through quickstart Stage B3 steps 1-7 plus the manual visual checklist (four colours, badges, walls/blocks/bombs/flames, smooth chain explosions, resize); record the result in `specs/002-web-client/checklists/manual-web.md` (create it with the checklist items ticked or noted). Commits: `render: add isometric projection, depth order, cubes and wasm exports`, `web: add http server, origin policy and websocket bridge`, `web: add the browser page and server binary`

**Checkpoint**: a browser can join and play a full round on a local server; origins are refused by default

---

## Phase 4: User Story 2 - Play browser, terminal and bot players together (Priority: P2)

**Goal**: one match with browser, terminal and bot participants that all see the same game and result (FR-005, FR-011, SC-002, SC-006)

**Independent Test**: `cargo test -p eksplode-re-web --test mixed`, then the manual mixed match in quickstart

### Tests for User Story 2 (write first, must fail)

- [X] T054 [P] [US2] Mixed-match tests in `crates/web/tests/mixed.rs`: `us2_as1_same_start` (one WebSocket client, one TCP `net::Client`, one bot; both receive the same `Start` seed and identical first-snapshot arena), `us2_as2_same_result` (run to the round limit with idle inputs and compare final observations ignoring `me` and the `Ended` outcome; repeat 5 times for SC-002)
- [X] T055 [P] [US2] Disconnect tests in `crates/web/tests/mixed.rs`: `us2_as3_closing_the_page_gives_a_bot_takeover` (WebSocket client closes mid-match; the TCP client receives `PlayerLeft{replaced_by_bot: true}` and ticks keep arriving), a silent WebSocket client is dropped after the shortened silence timeout, abrupt TCP reset of a browser connection behaves the same (SC-006 over 10 repetitions)
- [X] T056 [P] [US2] Capacity tests in `crates/web/tests/mixed.rs`: `us2_as4_a_fifth_participant_is_refused` with 2 WebSocket + 2 TCP clients filling the match, then one more WebSocket and one more TCP client both get `Rejected{Full}`; two tabs from the same address count as two players; duplicate display names are allowed and shown with their slot number
- [X] T057 [P] [US2] Page tests in `crates/web/js-tests/app.test.mjs`: the reducer shows `Player N left, a bot took over` for `PlayerLeft`, keeps rendering after it, shows `Full` refusal text, and shows the owner name (`only <name> can start`) when `NotOwner` arrives

### Implementation for User Story 2

- [X] T058 [US2] Make T054-T056 pass: fix any bridge issue they reveal in `crates/web/src/ws.rs`/`bridge.rs` (for example forwarding order, closing the in-process client on socket close so the host sees `Gone`, treating abrupt resets as disconnects)
- [X] T059 [US2] Add the reducer cases of T057 to `crates/web/assets/app.js` and show the slot's controller type (you, browser/terminal player, bot) in the HUD using the `Lobby` message, with the colour chip and number badge matching the cubes
- [X] T060 [US2] Run quickstart "Mixed match" manually (browser tab + terminal client + bot), then gates. Commits: `web: cover mixed browser and terminal matches`, `web: show other players and takeovers in the page`

**Checkpoint**: US1 and US2 complete; all client types are peers

---

## Phase 5: User Story 3 - Launch and join from the command line (Priority: P3)

**Goal**: single-command server start, terminal join by address, host-from-terminal with the web page, and the lobby start fix verified (FR-006, FR-007, FR-008, SC-008)

**Independent Test**: `cargo test -p eksplode-re-cli -p eksplode-re-web`, then quickstart Stage C

### Tests for User Story 3 (write first, must fail)

- [X] T061 [P] [US3] CLI argument tests in `crates/cli/tests/args.rs`: `host` accepts `--web-port N` (`1-65535`) and `--bind IP`; defaults unchanged (`port 7878`, `bots 0`, name `Host`); invalid ports and IPs are usage errors (exit 2); `join` accepts an `http://` or `ws://` style address by stripping the scheme and keeping the host (documented as a convenience) and still adds `:7878` when no port is given
- [X] T062 [P] [US3] Host banner tests in `crates/cli/tests/online.rs`: `us3_as1_` the hosting lobby shows `Web page: http://` only when `--web-port` was given, the terminal join address, and how to quit; with a non-loopback `--bind` the exposure `WARNING:` is printed to stderr (function `host_warnings(&HostOptions)` returns the lines)
- [X] T063 [P] [US3] Join error tests in `crates/cli/tests/online.rs`: `us3_as3_` refused connection, `Rejected{Full}`, `BadVersion`, `BadName`, `InMatch` each produce a one-line explanation and exit status 1 through `app::run_join` error mapping; a successful join lands in the same lobby as a browser player (`crates/web/tests/mixed.rs` adds `us3_as3_cli_join_sees_browser_player`)
- [X] T064 [P] [US3] Port-in-use test in `crates/web/tests/lifecycle.rs`: `us3_as2_` `eksplode-re-server --http-port <busy>` exits with status 1 and a message containing the port number (spawn the built binary with `env!("CARGO_BIN_EXE_eksplode-re-server")`)
- [X] T065 [P] [US3] Enter regression through the real binary in `crates/cli/tests/pty.rs` (extends T014): `us3_as4_` for `eksplode-re-cli host` and for `eksplode-re-cli join` to a headless `eksplode-re-server`, the first joiner presses CR then LF variants and the match starts every time (SC-008 over 20 attempts)

### Implementation for User Story 3

- [X] T066 [US3] Extend `crates/cli/src/args.rs` with `HostOptions { web_port: Option<u16>, bind: IpAddr }` per T061 and update the `USAGE` text and `contracts/cli-and-server-options.md` examples if wording changes
- [X] T067 [US3] Make `crates/cli` depend on `eksplode-re-web` and start the web server in `app::run_host` when `--web-port` is set: it reuses the same `net::Host` through `eksplode_re_web::Server::attach(host, options)` (add `Server::attach` in `crates/web/src/lib.rs` so one process shares one host), with the same origin policy and warnings; shutdown on quit
- [X] T068 [US3] Implement `host_warnings` and the lobby banner lines in `crates/cli/src/app.rs` and `crates/cli/src/online.rs` (T062); print warnings to stderr before the terminal switches to the alternate screen
- [X] T069 [US3] Map join failures to clear messages and exit code 1 in `crates/cli/src/app.rs` (T063) and make `Online` keep the `Rejected` notice on screen for 2 s before `run_join` returns the error (so the user can read it)
- [X] T070 [US3] Run T061-T065, `python3 scripts/pty-enter-check.py`, quickstart Stage C by hand, then gates. Commits: `cli: host with the web page, bind option and clearer join errors`, `web: let the terminal host share its match with the page`

**Checkpoint**: US1-US3 complete; the whole run-and-join story works from the command line

---

## Phase 6: User Story 4 - Understand the project from its documentation (Priority: P4)

**Goal**: README and documentation explaining the architecture and how everything works (FR-015, SC-009)

**Independent Test**: `cargo test -p eksplode-re-web --test docs`, then a fresh reader follows the README quick start

### Tests for User Story 4 (write first, must fail)

- [X] T071 [P] [US4] Docs consistency tests in `crates/web/tests/docs.rs`: `us4_as1_` README contains a Quick start section with the commands `cargo run -p eksplode-re-web` and `cargo run -p eksplode-re-cli -- join`; `us4_as2_` every directory under `crates/` is mentioned by name in `docs/ARCHITECTURE.md`; every flag in `eksplode-re-cli --help` and `eksplode-re-server --help` output appears in `README.md` or `docs/WEB.md`; every `ClientMsg` and `ServerMsg` variant name appears in `docs/PROTOCOL.md`; `us4_as3_` `docs/SECURITY.md` and `docs/WEB.md` contain the words `Origin`, `CORS`, `encryption` and `authentication`; every relative link in `README.md` and `docs/*.md` points to an existing file; `docs/ARCHITECTURE.md` contains a mermaid code block
- [X] T072 [P] [US4] Doc freshness test in `crates/web/tests/docs.rs`: the options table in `docs/WEB.md` is generated from the server's `USAGE` constant (a test regenerates it and fails with a diff if the committed table differs) so it cannot drift

### Implementation for User Story 4

- [X] T073 [US4] Rewrite `README.md`: what the project is, feature list, screenshots placeholder with a note, Quick start (5 commands: build, `cargo run -p eksplode-re-web -- --bots 1`, open the URL, join from a terminal with `cargo run -p eksplode-re-cli -- join 127.0.0.1`, run tests), modes (local, bots, host/join, server + web), controls for terminal and browser, safety summary with links, repository map, documentation index, commit convention (T071 `us4_as1_`)
- [X] T074 [P] [US4] Write `docs/ARCHITECTURE.md`: goals and principles (rules in `core`, thin clients, determinism), the mermaid component diagram from the plan, one section per crate (`core`, `bots`, `net`, `cli`, `web`, `web-render`) with responsibilities and key types, the tick loop and step order, the life of a key press (browser key -> `Input` JSON -> bridge -> host -> `step` -> `Snapshot` -> WASM -> canvas) as a mermaid sequence diagram, how bots plug in (`Observation` -> `PlayerInput`), lobby ownership, disconnect handling, testing strategy and where tests live, extension points (add a rule, a power-up, a bot behaviour, a screen) (T071 `us4_as2_`)
- [X] T075 [P] [US4] Write `docs/PROTOCOL.md`: transports (TCP lines, WebSocket text frames), every `ClientMsg` and `ServerMsg` with fields and an example, error codes, limits (64 KiB, 10 errors, 10 s silence), ordering and ownership rules, versioning (additive changes keep version 1); move the content of `specs/001-bomberman-multiplayer/contracts/network-protocol.md` here and update the README link that pointed to the planning workspace (T071)
- [X] T076 [P] [US4] Write `docs/WEB.md`: server options table (generated per T072), routes and headers, origin and CORS policy with examples (`--allow-origin`, `--allow-any-origin`), the page structure (`index.html`, `app.js`, `style.css`), the WASM module and its exports with the command layout, how to rebuild it (`rustup target add wasm32-unknown-unknown`, `scripts/build-wasm.sh`), how to debug in the browser console, running the JS tests (T071, T072)
- [X] T077 [P] [US4] Write `docs/SECURITY.md`: no encryption and no authentication, loopback default bind, what exposing to a network means, CORS/Origin policy and why, request and message limits, what is not protected (no spectator isolation, no rate limiting beyond limits), safe hosting tips (firewall, SSH tunnel, reverse proxy with TLS) (T071 `us4_as3_`)
- [X] T078 [P] [US4] Write `docs/DEVELOPMENT.md`: prerequisites, build, test (including `node`, `python3` helpers), lint and format gates, commit convention and hook installation, repository layout, how to add a rule with a test first, how to add a bot behaviour, how to add a page feature, release checklist (rebuild and commit the wasm)
- [X] T079 [US4] Follow the README quick start literally on a clean checkout (`git clone game /tmp/eksplode_re-check`, build, run server, join from browser and terminal), fix every gap, and record the elapsed steps in `specs/002-web-client/checklists/manual-web.md` (SC-001, SC-009); ask one reader (or a fresh review pass) the five architecture questions of SC-009 and record the answers. Run T071-T072 and gates. Commits: `docs: rewrite README with quick start and repository map`, `docs: add architecture, protocol, web, security and development guides`

**Checkpoint**: all four stories complete and documented

---

## Phase 7: Polish & Cross-Cutting Concerns

- [X] T080 [P] Add `scripts/check-scenarios.sh` verifying that every acceptance scenario id of this feature (`us1_as1`-`us1_as6`, `us2_as1`-`us2_as4`, `us3_as1`-`us3_as4`, `us4_as1`-`us4_as3`) appears in a test name or in `specs/002-web-client/checklists/manual-web.md`; run it in CI (FR-016, SC-010)
- [X] T081 [P] Audit non-test code in `crates/web` and `crates/web-render` for `unwrap`, `expect`, `panic!`, unchecked indexing and unchecked arithmetic on external data (headers, frame sizes, WASM lengths) and replace with typed errors; add a fuzz-style test feeding random bytes to the HTTP parser and to the WebSocket bridge decoder (constitution rule on external input)
- [X] T082 [P] Verify performance criteria and record results in `docs/DEVELOPMENT.md` and `specs/002-web-client/checklists/manual-web.md`: p95 key-to-snapshot latency below 200 ms with 2 WebSocket + 2 TCP clients (extend `crates/net/tests/consistency.rs` style test in `crates/web/tests/mixed.rs`), wasm size <= 300 KB, page total <= 500 KB, 60 fps feel during chain explosions (SC-003, SC-005)
- [X] T083 [P] Verify SC-007 and SC-008 explicitly: 100 cross-origin upgrade attempts with default settings all get 403 (test in `crates/web/tests/origin.rs`), and the 20-attempt Enter check of T065 passes
- [X] T084 Update `specs/001-bomberman-multiplayer/contracts/cli-interface.md` and `quickstart.md` to the real behaviour (bots 0-4, P4 keys TFGH only, default bots fill to 4, `--lobby`, host/join, no native GUI) with a note pointing to feature 002, and mark the stage-4 GUI wording as replaced
- [X] T085 Run the full gates (`cargo fmt --check`, `cargo clippy --workspace --all-targets -- -D warnings`, `cargo test --workspace`, `./scripts/build-wasm.sh`, `node --test crates/web/js-tests`, `python3 scripts/pty-enter-check.py`, `scripts/check-scenarios.sh`) and the whole quickstart.md once more; fix gaps
- [X] T086 Review the `game/` history against the commit convention (`git log --format=%s | awk 'length>80'` is empty and every subject matches the hook). Commit: `docs: finalise web client notes and success criteria results`

---

## Dependencies & Execution Order

### Phase Dependencies
- **Setup (1)** then **Foundational (2)** block everything (the lobby owner commands and `Host::connect` are used by the web bridge, and FR-017 requires the Enter fix first).
- **US1 (3)** depends on Phase 2. Inside US1: render module (T022-T027, T037-T042) and server (T028-T032, T043-T049) are independent tracks; the page (T050-T052) needs the wasm artifact (T042) and the server routes (T046).
- **US2 (4)** depends on US1 (bridge and page). **US3 (5)** depends on Phase 2 and on the server library from US1 (`Server::attach`). **US4 (6)** should come last because it documents final behaviour, but T074-T078 can start once US1 is stable.
- **Polish (7)** after all desired stories.

### Within a story
- Tests first and failing, then types, then logic, then page/CLI, then manual check and commit.
- US1: T037/T038/T039 parallel, then T040, then T041, then T042. T043/T044/T045 parallel, then T046, then T047, then T048, then T049. T050 parallel with the server track; T051 then T052.

### Parallel Opportunities
- Setup: T004, T005, T006
- Foundational tests T009-T014 all parallel; implementation T015/T016 parallel, then T017, T018, T019, T020
- US1 render tests T022-T027 and server tests T028-T032 and page tests T033-T036 all parallel (different files); T037-T039, T043-T045 parallel
- US2 tests T054-T057 parallel
- US3 tests T061-T065 parallel
- US4 docs T074-T078 parallel; T071/T072 first

### Parallel Example: User Story 1
```text
render track:  T022 T023 T024 T025 T026 T027 (tests)  ->  T037 T038 T039  ->  T040 -> T041 -> T042
server track:  T028 T029 T030 T031 T032      (tests)  ->  T043 T044 T045  ->  T046 -> T047 -> T048 -> T049
page track:    T033 T034 T035 T036 (tests)   ->  T050 (parallel)  ->  T051 -> T052   (needs T042, T046)
then T053 manual walkthrough
```

## Implementation Strategy

### MVP First (Foundational + US1)
1. Phases 1-2: the Enter fix is a user-visible win by itself; validate with `scripts/pty-enter-check.py`.
2. Phase 3: stop and validate with quickstart Stage B3 (a browser plays a full round on a local server).
3. This is the first demonstrable product of feature 002.

### Incremental Delivery
1. Enter fix and owner commands, then US1 browser play, then US2 mixed play, then US3 CLI options, then US4 documentation.
2. Each stage ends with all gates green and commits in the `part: short desc` style.

### Notes
- `FR-xxx`/`SC-xxx`/`US-AS` ids are referenced in tasks for traceability.
- Never skip the red step: run the new tests and see them fail before implementing.
- Rebuild and commit `crates/web/assets/eksplode_re_render.wasm` whenever `crates/web-render` changes (CI verifies it builds).
- Ctrl+C handling (T049) adds the `ctrlc` crate as the second new dependency of this feature; the plan's Complexity Tracking should list it.
