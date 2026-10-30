# Contract: Room Setting `bot_difficulty`

Additive change to the room settings object used by v2 `CreateRoom` / `SetRoom` / `RoomUpdated` /
`RoomSummary` and by the legacy v1 `SetRoom` / `Lobby` messages (see `game/docs/PROTOCOL.md`,
"Room settings"). No protocol version bump.

## Settings object

```json
{"name":"Friday room","mode":"normal","size":"medium","minutes":5,"friendly_fire":true,
 "bot_difficulty":"medium"}
```

| Field | Values | Default |
| --- | --- | --- |
| `bot_difficulty` | `"easy"`, `"medium"` or `"hard"` | `"medium"` |

## Rules

1. Missing field → `"medium"`; any other value or type (`"HARD"`, `3`, `null`, `[]`) → `"medium"`.
   Never an error, never closes the connection.
2. Servers always send the field in settings they emit.
3. Changing it uses the existing `SetRoom`: owner only (`NotOwner` otherwise), lobby phase only
   (`WrongPhase` / `InMatch` otherwise), revision-checked as today.
4. The value at `Start` applies to every bot of that match, including bots that take over a seat
   mid-match. Behaviours are never sent on the wire.

## Command-line flags

| Binary | Flag | Values | Default |
| --- | --- | --- | --- |
| `eksplode-re-cli host` | `--bot-difficulty <D>` | `easy`, `medium`, `hard` | `medium`; counts as a room-setting flag (skips setup like `--mode`) |

An invalid value is a usage error (`UsageError`), consistent with `--friendly-fire`. `local` and
`eksplode-re-web` have no such flag: only the host chooses the difficulty.

## Developer command `DevSetBotBehaviour`

| Protocol | Message | Fields |
| --- | --- | --- |
| v2 request | `DevSetBotBehaviour` | `request_id`, `room_id`, `behaviour: "aggressive" \| "protective" \| "balanced" \| null` |
| legacy client | `DevSetBotBehaviour` | `behaviour` as above |
| reply / broadcast | `DevState` | existing `paused`, `drops`, plus `behaviour` (name or `null`; missing = `null`) |

- `null` = Random (default). Owner only (`NotOwner`), developer mode only (`Unsupported`; legacy
  host `DevDisabled`), unknown name or type → `BadRoom` (legacy host `BadValue`). Allowed in the
  lobby and during a match; applies to bots created
  afterwards (next `Start`, takeovers). Broadcast to everybody as `DevState`.
