# Research: Web Client and Project Documentation

All items from the Technical Context are resolved; no NEEDS CLARIFICATION remain.

## R1. Why Enter "does not work" (bug analysis)
- **Decision**: Fix three causes together: (a) normalise key events so Enter, CR, LF, Ctrl+J and Ctrl+M all mean Enter and modified keys are never mistaken for plain letters; (b) let the lobby owner start from any client with new `Start`, `AddBot`, `RemoveBot` messages; (c) answer every refused lobby command with a visible `Error` shown as a notice.
- **Evidence**: pseudo-terminal harness (`scripts/pty-enter-check.py`): CR/CRLF/kitty start the match; LF does not; Enter in a joined client does nothing; refusals are silent.
- **Rationale**: matches the reported symptom ("CLI connection not working on Enter") for both host and joined sessions and satisfies FR-008/SC-008, including "a failure to start MUST be shown".
- **Alternatives**: only handle LF (leaves clients unable to start and silent failures); require "ready" from everyone before start (changes game flow, not asked).

## R2. HTTP and WebSocket server stack
- **Decision**: std `TcpListener` and threads for HTTP (tiny parser: request line, headers, size limits, timeouts) plus `tungstenite` (blocking) for WebSocket. The upgrade request is peeked (`TcpStream::peek`) so non-WebSocket requests are answered by our code and upgrades are handed to `tungstenite::accept_hdr`, whose callback performs the `Origin` check.
- **Rationale**: consistent with R6 of feature 001 (no async runtime); only routes are five fixed embedded files plus `/ws`; small attack surface; blocking IO with a 20 ms read timeout lets one thread per socket pump both directions.
- **Alternatives**: `axum`/`tokio` (async runtime, large dependency tree); `tiny_http` (no WebSocket upgrade support); hand-written WebSocket framing (security and correctness risk).

## R3. WebAssembly module
- **Decision**: crate `web-render` as `cdylib` + `rlib`, target `wasm32-unknown-unknown`, **no wasm-bindgen**. Exports: `eksplode_re_alloc(len)`, `eksplode_re_free(ptr,len)`, `eksplode_re_load(ptr,len)->i32` (parse an `Observation` JSON), `eksplode_re_frame(w,h,alpha)->u32` (build commands, return count), `eksplode_re_output_ptr()->ptr`. Command stride is 16 `f32`s. `ffi.rs` is the only file allowing `unsafe`.
- **Rationale**: only `rustup target add wasm32-unknown-unknown` is needed; JSON parsing reuses `core`'s `Observation` with `serde`, so the page and server share types; everything but the three FFI helpers is safe Rust testable natively.
- **Build**: `scripts/build-wasm.sh` runs `cargo build -p eksplode-re-web-render --target wasm32-unknown-unknown --profile wasm-release` (opt-level `z`, LTO, `panic=abort`, strip) and copies to `crates/web/assets/eksplode_re_render.wasm`; the artifact is committed and CI rebuilds it to verify it compiles. Size budget 300 KB.
- **Alternatives**: wasm-bindgen/trunk (extra CLI tools, generated unsafe); pure JS rendering (ignores the "WASM" request); drawing inside WASM with `web-sys` canvas bindings (needs bindgen).

## R4. Origin and CORS policy
- **Decision**: serve page and WebSocket from the same origin; on upgrade compare `Origin` with the request's own scheme+`Host`; mismatch or a missing `Origin` from a browser-like request is refused with `403`. `--allow-origin ORIGIN` (repeatable, exact match) and `--allow-any-origin` relax it; each prints `WARNING: cross-origin access enabled for ...` at start-up and static responses then carry `Access-Control-Allow-Origin`. Non-browser clients without `Origin` (e.g. test tools) are allowed only when the connection comes from loopback.
- **Rationale**: matches the user's "warning of CORS" wish and spec FR-009/SC-007; cross-site pages cannot silently drive a local game server.
- **Alternatives**: always `*` (unsafe); no check (any web page could connect to a LAN server from a visitor's browser).

## R5. Bind address, exposure and address printing
- **Decision**: default bind `127.0.0.1` for HTTP and game ports; `--bind 0.0.0.0` exposes to the network and prints the warning "no encryption, no authentication, anyone on the network can play". Printed addresses: loopback plus the primary LAN address found with the UDP-connect trick (no packet is sent); other interfaces are reachable with the same port.
- **Rationale**: safe default, explicit opt-in; std only (no interface-enumeration crate).
- **Alternatives**: bind all interfaces by default (surprising exposure); enumerate all interfaces with a crate (extra dependency for a convenience message; documented as a limitation).

## R6. Lobby owner and start protocol
- **Decision**: `Lobby::owner()` = lowest slot with `Local` or `Remote` controller. New additive `ClientMsg` variants `Start`, `AddBot`, `RemoveBot`; the host checks owner, phase and participant count and replies with `Error{NotOwner|CannotStart|WrongPhase}` or the usual `Lobby`/`Start` broadcast. `PROTOCOL_VERSION` stays 1 (old clients simply never send them).
- **Rationale**: a headless server has no local player; owner derivation needs no extra state or message field and survives the owner leaving.
- **Alternatives**: separate admin token (needs UI to enter it); everybody may start (griefing); ready-vote (slower, more states).

## R7. Isometric rendering (in `web-render`)
- **Decision**: tile size 64x32 (2:1 dimetric), `sx = (x - y) * 32`, `sy = (x + y) * 16 - z`; floor tiles first, then objects sorted by `(x + y, layer)` (painter's algorithm); walls and blocks are 3-face boxes; players are cubes (top 1.0, left 0.8, right 0.6 brightness) with a number badge; bombs are circles that pulse with the fuse; flames are translucent diamonds; power-ups are small rotating diamonds with a letter. Palette: red `#dc322f`, blue `#268bd2`, green `#3cb44b`, yellow `#e6be1e` (pairwise RGB distance > 100). View scaled to fit the canvas, centred. Interpolation: positions lerp between the previous and current tick by `alpha`.
- **Rationale**: simple, deterministic, unit-testable; number badge helps colour-blind users (SC-004).
- **Alternatives**: WebGL (heavier, not needed at this scale); true 3D engine (out of scope).

## R8. Keyboard and timing in the page
- **Decision**: `keydown`/`keyup` maintain a held-key set; every 50 ms (20 Hz) `app.js` sends `Input{tick, mv, bomb}` where `mv` is the most recently pressed held direction and `bomb` is a one-shot flag; `blur` and `visibilitychange` clear the set. Arrow keys + Space and WASD + E both work (one player per browser). Rendering runs on `requestAnimationFrame` with `alpha = (now - lastSnapshot) / tickMs`.
- **Rationale**: real key-up information makes held movement exact (unlike the terminal); clearing on blur satisfies the edge case.
- **Alternatives**: send only key transitions (needs more server state); rely on key repeat (laggy start).

## R9. Disconnects, background tabs and reconnect
- **Decision**: WebSocket close or silence beyond the host's 10 s timeout triggers the existing bot takeover. Background tabs keep their connection; `app.js` sends `Ping` every 2 s while hidden so timers throttled to 1 s do not look silent. "Reconnect" opens a new socket and joins again as a new player (the old slot is a bot); the UI says so.
- **Rationale**: reuses host logic unchanged; avoids session tokens.
- **Alternatives**: resume tokens (more protocol and security surface).

## R10. Testing strategy
- **Decision**: (a) `web-render` unit tests natively (projection, depth, cube, palette, interpolation, command counts, FFI round trip); (b) `node --test` loads the committed `.wasm` and checks the command buffer for a sample observation and `app.js` pure helpers (key state to input, message handling); (c) Rust socket tests: static routes and headers, size limits, 404/405, origin accept/refuse/warn, WebSocket join/start/play, mixed browser+terminal match with identical outcome, malformed/oversized frames, disconnect with bot takeover; (d) pty regression test for the Enter keys; (e) manual browser checklist for visuals; (f) docs consistency test.
- **Rationale**: automated coverage of everything except actual pixels, which get a short manual checklist plus an optional screenshot when a browser is available.

## R11. Documentation set
- **Decision**: `README.md` (what, quick start in 5 commands, modes, controls, safety, tests, repository map) and `docs/` with ARCHITECTURE (mermaid diagram, crate responsibilities, tick loop, message flow key press to screen, bot plugging, extension points), PROTOCOL (messages, transports, errors, examples), WEB (server flags, routes, origin policy, WASM build, page structure), DEVELOPMENT (build/test/lint/commit style, adding a rule, a bot behaviour, a screen), SECURITY. A docs test checks that every crate directory and every CLI/server flag appears in the documentation.
- **Rationale**: satisfies FR-015/SC-009 with something that cannot silently rot.

## R12. Closing the native GUI work
- **Decision**: mark T070-T087 of feature 001 as replaced by feature 002 in its `tasks.md`; revert the unfinished `tick_with` refactor in `crates/cli/src/app.rs`; drop the macroquad dependency idea.
- **Rationale**: avoids dead code and half-built crates; spec 002 declares the replacement.
