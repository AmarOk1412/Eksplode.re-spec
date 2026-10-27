# Contract: Room settings and protocol (version 1, additive)

## Messages

### Client to host

| Message | JSON | When | Notes |
|---------|------|------|-------|
| `RoomQuery` | `{"type":"RoomQuery"}` | before or after `Join` | answered with `RoomInfo`; allowed before `Join`, does not join; does not count against the join timeout |
| `Join` (extended) | `{"type":"Join","version":1,"name":"...","look":{...},"room":{...}}` | first message of a player | `room` is optional; used only if this join creates the room |
| `SetRoom` | `{"type":"SetRoom","settings":{...}}` | lobby phase, owner only | replaces the settings after sanitizing; every connection gets a fresh `Lobby` |

### Host to client

| Message | JSON | Meaning |
|---------|------|---------|
| `RoomInfo` | `{"type":"RoomInfo","open":false,"settings":{...},"humans":0}` | state of the room; `settings` are the start-up defaults when closed |
| `Lobby` (extended) | `... "settings":{...}` | settings visible to everybody |
| `Welcome` (extended) | `... "settings":{...}` | same value as in `Lobby` for the first display |

### Settings object

```json
{"name":"Friday room","mode":"team","size":"medium","minutes":5,"friendly_fire":false}
```

Decoding rules (all lenient, see `net/src/room.rs`):

- not an object: defaults;
- unknown fields: ignored;
- missing or wrong-typed field: its default;
- `mode`/`size` unknown string: default; `minutes` not in {2, 3, 5, 10}: 3;
- `name`: control characters removed, trimmed, cut to 24 characters; empty becomes "<owner>'s room" (host side).

## Error codes (new)

| Code | When |
|------|------|
| `BadRoom` | `SetRoom` whose `settings` is not an object |
| existing `NotOwner` | `SetRoom` from a non-owner |
| existing `WrongPhase` | `SetRoom` during a match |
| existing `CannotStart` | `Start` when the mode's player count is not met (the message says how many are missing; the owner can add bots) |

## Rules for the host

1. `RoomQuery` is answered immediately without consuming a slot.
2. A `Join` that finds the room closed creates it: the owner is that player, the settings are `Join.room` sanitized or the start-up defaults.
3. `SetRoom` takes effect at once and is broadcast through `Lobby`; a settings change also clears "ready" states if any exist (none in this version).
4. At `Start`, `Config = settings.to_config(base)`; the seed comes from the host option or the clock as before.
5. When the last human disconnects in the lobby, the room closes; during a match the existing disconnect rules apply and the room closes when the match is left.
6. Messages stay at most 64 KiB; settings messages are far below.

## Compatibility

- An old client never sends `RoomQuery`/`SetRoom`; it joins and plays in the room as before; a `Lobby` with `settings` is ignored by it.
- A new client talking to an old host gets no `RoomInfo` (the page shows the old join screen after a short wait of 1 second).
- Protocol version stays 1.

## Tests

`f4_us1_as1_*` .. `f4_us1_as8_*` in `net/tests`: closed/open room info, creating with `Join.room`, ignoring `room` when open, owner-only `SetRoom`, wrong phase, hostile settings (wrong types, huge name, control characters, nested junk), 64 KiB limit, old-client sequence unchanged.
