# Contract: Network Protocol (Stage 3)

> The maintained version of this contract is `game/docs/PROTOCOL.md` (it adds `Start`, `AddBot`,
> `RemoveBot`, the new error codes and the WebSocket transport). This file is kept as the
> original planning record.

Transport: TCP, default port 7878. Encoding: one JSON object per line (`\n` terminated), UTF-8,
max 64 KiB per line. Every message is an object `{"type": "<Variant>", ...fields}`.
Protocol version: `1`. The host is authoritative.

## Client -> Host

| type | Fields | When |
|------|--------|------|
| `Join` | `version: u32`, `name: string` (1-16 chars) | first message |
| `SetReady` | `ready: bool` | in lobby |
| `Input` | `tick: u32`, `mv: "Up"/"Down"/"Left"/"Right"/null`, `bomb: bool` | during play, at most one per tick |
| `Leave` | - | graceful exit |
| `Ping` | `nonce: u64` | optional keepalive |

## Host -> Client

| type | Fields | When |
|------|--------|------|
| `Welcome` | `slot: 0..3`, `config`, `lobby` | accepted join |
| `Rejected` | `reason: "Full" / "BadVersion" / "BadName" / "InMatch"` | join refused (FR-013) |
| `Lobby` | `slots: [{controller, name, ready}; 4]` | any lobby change |
| `Start` | `seed: u64`, `config` | host starts match |
| `Snapshot` | `tick: u32`, `state` (Observation) , `events: [Event]` | each tick during play |
| `PlayerLeft` | `slot`, `replaced_by_bot: bool` | disconnect (FR-016) |
| `Ended` | `outcome` | round over |
| `Error` | `code`, `message` | invalid message (FR-015) |
| `Pong` | `nonce` | reply to `Ping` |

## Rules
- A connection must send `Join` first; any other first message gets `Error` and a close.
- Unknown `type`, bad JSON, oversize line, wrong phase, or inputs for another slot -> `Error`, message dropped, match unaffected. After 10 errors the connection is closed.
- Input for tick `t` is applied at the next host tick if received before it; late inputs are dropped; a missing input means "no action".
- A 5th `Join` gets `Rejected{Full}` and is closed.
- No input and no ping for 10 s while playing counts as a disconnect.
- Host leaving: remaining clients get `Ended` + close with an `Error{code:"HostLeft"}`.

## Test obligations
Loopback tests: join/welcome, full match refuses 5th, malformed line is rejected and others continue, disconnect -> bot takeover, all clients receive identical final `Ended`.
