# Data Model: Web Client and Project Documentation

Entities from the spec and the types that carry them. Existing types (`Lobby`, `Slot`, `Controller`, `Observation`, `Event`, `ServerMsg`, `ClientMsg`) are unchanged unless listed here.

## Changes to existing types

### `core::Lobby`
- **New method** `owner() -> Option<usize>`: lowest slot whose controller is `Local` or `Remote`; `None` when only bots/empty.
- **New method** `last_bot() -> Option<usize>`: highest slot holding a `Bot` (used by `RemoveBot`).

### `net::ClientMsg` (additive; protocol version stays 1)
| Variant | Fields | Who may send | Effect |
|---------|--------|--------------|--------|
| `Start` | - | lobby owner, lobby phase, >= 1 participant | starts the match (same as `Host::start_match`) |
| `AddBot` | - | lobby owner, lobby phase, free slot | adds a bot to the first free slot |
| `RemoveBot` | - | lobby owner, lobby phase | frees the highest bot slot |

Refusals are `ServerMsg::Error { code, message }` with codes:

| code | When |
|------|------|
| `NotOwner` | sender is not the lobby owner |
| `CannotStart` | no participant, or a match is already running |
| `WrongPhase` | command sent while a match runs (add/remove bot, ready) |
| `NoSlot` | `AddBot` with all slots taken |

### `net::Host`
- **New** `connect(local: bool) -> Client`: an in-process connection that has not joined yet (used by the WebSocket bridge; `local_client` becomes `connect(true)` + `Join`).

### `cli` key handling
- **New** `input::normalize(KeyEvent) -> KeyCode`: `Enter`, `Char('\r')`, `Char('\n')`, `Ctrl+J`, `Ctrl+M` become `Enter`; other keys with Ctrl/Alt are ignored instead of acting as plain letters.
- `Online::lobby_action` returns an `Option<String>` notice for refused actions and shows server `Error` messages.

## New entities

### Server options (`web::ServerOptions`)
| Field | Type | Default | Validation |
|-------|------|---------|------------|
| `bind` | IP address | `0.0.0.0` | valid IP; `0.0.0.0`/non-loopback triggers the exposure warning |
| `http_port` | u16 | 8080 | 1-65535; `0` allowed for tests (ephemeral) |
| `game_port` | u16 | 7878 | 1-65535; `0` allowed for tests |
| `bots` | u8 | 0 | 0-3 |
| `seed` | Option<u64> | none | - |
| `origins` | `AllowedOrigins` | `SameOrigin` | see below |
| `max_connections` | usize | 64 | >= 4 |

### `AllowedOrigins`
- `SameOrigin` (default) | `List(Vec<String>)` (exact `scheme://host[:port]` matches, same-origin always allowed) | `Any`.
- `List`/`Any` produce a warning line at start-up; `Any` adds a stronger one.

### Web connection (`web::ws::Connection`, one per browser tab)
| Field | Meaning |
|-------|---------|
| `client` | in-process `net::Client` created by `Host::connect(false)` |
| `errors` | malformed frames so far; closed at 10 (mirrors the TCP rule) |
| `last_seen` | time of the last frame (the host enforces the 10 s silence rule) |

States: `Handshaking -> Open -> Closing -> Closed`. A refused origin never reaches `Open` (HTTP 403).

### HTTP request limits
Request head <= 8 KiB, read timeout 5 s, methods `GET`/`HEAD` only, paths: `/`, `/index.html`, `/app.js`, `/style.css`, `/eksplode_re_render.wasm`, `/ws`; WebSocket message <= 64 KiB (same as TCP lines).

### Draw command (WASM output, 16 `f32` per command)
| Index | Field | Meaning |
|-------|-------|---------|
| 0 | `kind` | 0 filled quad, 1 filled circle, 2 text, 3 filled triangle |
| 1-4 | `r,g,b,a` | colour 0.0-1.0 |
| 5-12 | `x0,y0 .. x3,y3` | quad/triangle points in canvas pixels (circle uses x0,y0; text uses x0,y0) |
| 13 | `size` | circle radius or font size in pixels |
| 14 | `text` | for kind 2: ASCII code of one character (digits, `B`, `R`, `S`) |
| 15 | `layer` | informational depth key (commands are already sorted) |

### Render state (inside the WASM module)
- `current: Observation`, `previous_positions: [Pos; 4]`, `current_positions: [Pos; 4]`, `tick: u32`.
- `load()` replaces `current`; when `tick` changes, previous := current positions.
- `frame(w,h,alpha)` is pure with respect to that state: same state + arguments = identical command buffer.

### Palette and badges
Colours by slot: red, blue, green, yellow (see research R7); badge text is the slot number 1-4 drawn on the cube's top face.

### Documentation set (checked by a test)
`README.md`, `docs/ARCHITECTURE.md`, `docs/PROTOCOL.md`, `docs/WEB.md`, `docs/DEVELOPMENT.md`, `docs/SECURITY.md`. Required content (enforced by the docs test): every crate directory name, every flag of `eksplode-re-cli` and `eksplode-re-server`, every `ClientMsg`/`ServerMsg` variant, the word "Origin" in WEB and SECURITY.

## Validation rules mapped to requirements

| Rule | FR |
|------|----|
| Static server serves only the five fixed files, GET/HEAD | FR-001 |
| Join validation (name, version, full, in match) is the host's, unchanged | FR-002, FR-005 |
| Draw list contains one cube per alive player, distinct colour, badge | FR-003 |
| Key state to `Input` is 20 Hz, last pressed direction wins, bomb is one-shot | FR-004 |
| Server prints web URL, terminal address, stop hint | FR-006 |
| `Start`/`AddBot`/`RemoveBot` owner rules and error codes | FR-008 |
| Origin check on upgrade; warnings when relaxed | FR-009 |
| Exposure warning when bind is not loopback | FR-010 |
| WebSocket close or silence maps to bot takeover | FR-011 |
| Page shows refused/full/lost/server-gone notices with retry | FR-012 |
| Frames decoded with `net::codec`; >64 KiB, non-text or malformed are rejected | FR-013 |

## State transitions

Server instance: `Lobby -> Playing -> Lobby` (unchanged, see feature 001). Browser page: `Connecting -> Joining -> Lobby -> Playing -> Lobby`, with `Closed(reason)` reachable from any state and `Retry` returning to `Connecting`.
