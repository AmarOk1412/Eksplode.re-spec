# Contract: HTTP and WebSocket

Served by `eksplode-re-server` (crate `eksplode-re-web`). Default addresses: HTTP and WebSocket on `127.0.0.1:8080`; terminal clients on TCP `127.0.0.1:7878`.

## HTTP routes

| Method + path | Response | Notes |
|---------------|----------|-------|
| `GET /` and `GET /index.html` | `200 text/html; charset=utf-8` | the page |
| `GET /app.js` | `200 text/javascript` | ES module |
| `GET /style.css` | `200 text/css` | |
| `GET /eksplode_re_render.wasm` | `200 application/wasm` | WebAssembly graphics module |
| `GET /ws` with `Upgrade: websocket` | `101 Switching Protocols` | see below |
| `HEAD` of any route above | same headers, no body | |
| any other path | `404 text/plain` | |
| any other method | `405` with `Allow: GET, HEAD` | |
| request head > 8 KiB | `431` | connection closed |
| no complete request within 5 s | connection closed | |
| more than `max_connections` open | `503` | |

Response headers on every static response: `Cache-Control: no-cache`, `X-Content-Type-Options: nosniff`, `Referrer-Policy: no-referrer`, `Content-Security-Policy: default-src 'self'; script-src 'self' 'wasm-unsafe-eval'; style-src 'self'; connect-src 'self' ws: wss:; img-src 'self' data:`. With relaxed origins, `Access-Control-Allow-Origin: <origin>` (or `*` for `--allow-any-origin`) is added.

## WebSocket `/ws`

### Handshake and origin rule
- Standard RFC 6455 handshake, subprotocol none.
- The request's `Origin` header must be one of: the server's own origin (scheme `http` + the request's `Host` value), an origin given with `--allow-origin`, anything with `--allow-any-origin`.
- A mismatching `Origin` is answered with `403 Forbidden` and a plain-text reason; nothing is forwarded to the match.
- A request without `Origin` is accepted only from a loopback peer (command-line tools); otherwise `403`.
- Messages larger than 64 KiB close the connection (`1009`).

### Framing
One JSON message per **text** frame. Binary frames are rejected with an `Error{Malformed}` message. Ping/pong control frames are answered by the server. The JSON is exactly `net::protocol::ClientMsg` (browser to server) and `ServerMsg` (server to browser), documented in `docs/PROTOCOL.md`; examples:

```json
{"type":"Join","version":1,"name":"Ann"}
{"type":"Start"}
{"type":"Input","tick":42,"mv":"Left","bomb":false}
```
```json
{"type":"Welcome","slot":0,"config":{...},"lobby":{...}}
{"type":"Snapshot","tick":43,"state":{...Observation...},"events":[...]}
{"type":"Error","code":"NotOwner","message":"only Ann can start the match"}
```

### Additive client messages
`Start`, `AddBot`, `RemoveBot` (see data-model). Error codes: `NotOwner`, `CannotStart`, `WrongPhase`, `NoSlot`, plus the existing `Malformed`, `NotJoined`, `AlreadyJoined`, `BadTick`, `Timeout`, `HostLeft`.

### Lifecycle
1. Upgrade accepted, connection created with `Host::connect(false)`; the first message must be `Join`.
2. Frames flow both ways; every decoded `ClientMsg` is forwarded unchanged to the host.
3. Malformed frame: `Error{Malformed}` reply, counted; the tenth closes the socket (close code `1008`).
4. Socket closed by either side, or no message for 10 s during a match: host performs the normal disconnect handling (bot takeover and `PlayerLeft` to the others).
5. Server shutdown: every socket receives `Error{HostLeft}` and a close frame (`1001`).

### Tests that bind this contract
Static routes and headers; 404/405/431/503; origin accepted, refused, relaxed with warning; join, start by owner, play, result over WebSocket; refusal codes; binary and oversized frames; disconnect and bot takeover; mixed browser + terminal match ends with the same outcome for all.

> Feature 003 adds two static routes, `GET /look.js` and `GET /audio.js` (`200 text/javascript`), with the same headers; see `specs/003-customization-powers-audio/contracts/cli-and-server-options.md`.

## Feature 003 additions

Feature 003 adds the static routes `/look.js` and `/audio.js`, optional fields (`Input.punch`,
`Welcome.dev`, looks in `Welcome` and `Lobby`), the messages `SetLook`, `DevPauseBots`,
`DevSetDrops`, `DevResetDrops` and `DevState`, and the error codes `LookAdjusted`, `DevDisabled`
and `BadValue`. The source of these changes is
`specs/003-customization-powers-audio/contracts/protocol-changes.md` and
`cli-and-server-options.md`; protocol version 1 is unchanged.
