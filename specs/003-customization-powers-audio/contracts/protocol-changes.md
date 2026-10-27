# Contract: Protocol Changes (still version 1)

All changes are additive: new optional fields use defaults, new messages are ignored by clients that do not know them. The base contract is `game/docs/PROTOCOL.md`.

## Look on the wire

```json
{"colour":"Blue","gender":"Male","head":"Cap","face":"None"}
```

| Field | Allowed values |
|-------|----------------|
| `colour` | `Blue`, `Black`, `Red`, `Purple`, `Green`, `Orange`, `Yellow`, `Pink`, `Teal`, `White` |
| `gender` | `Male`, `Female` |
| `head` | `None`, `Cap`, `TopHat`, `Crown`, `Headphones` |
| `face` | `None`, `Glasses`, `Sunglasses`, `Moustache`, `Mask` |

Decoding is lenient (research R4): a missing field, a wrong type or an unknown value is replaced by the slot default for that field; a value that is not an object gives the whole default. A look never causes `Error{Malformed}`.

## Client to host

| type | Fields | Rules |
|------|--------|-------|
| `Join` | `version`, `name`, `look` (optional, loose JSON) | unchanged otherwise; the look is resolved for colour conflicts |
| `SetLook` | `look` (loose JSON) | lobby only (`Error{WrongPhase}` during a match); resolved for conflicts; the host answers with a `Lobby` broadcast |
| `Input` | `tick`, `mv`, `bomb`, `punch` (default `false`) | `punch` without the punch power is ignored |
| `DevPauseBots` | `paused: bool` | developer mode and owner only |
| `DevSetDrops` | `chances: [f32; 8]` in power order `ExtraBomb, Range, Speed, Punch, Slow, HyperSpeed, SmallBomb, Kick` | developer mode and owner only; each value finite and within 0..=1, otherwise `Error{BadValue}` and nothing changes |
| `DevResetDrops` | none | developer mode and owner only; restores the default table |

## Host to client

| type | Change |
|------|--------|
| `Welcome` | adds `looks: [Look; 4]` and `dev: bool` (both default to "absent": default looks, `false`) |
| `Lobby` | adds `looks: [Look; 4]` |
| `DevState` | new: `paused: bool`, `drops: [f32; 8]`; sent to everybody when it changes and to a joiner when `dev` is on |
| `Snapshot` | `state` (the observation) now carries `punch`, `malus` per player and `flight` per bomb; `events` include the new events |
| `Error` | new codes below |

## Error codes added

| code | Meaning |
|------|---------|
| `LookAdjusted` | the requested colour was taken; the message names the colour you got (a notice, not a refusal) |
| `DevDisabled` | a developer command on a server not started with `--dev` |
| `BadValue` | a developer value out of range or not finite |

`NotOwner`, `WrongPhase` keep their meaning for `Dev*` commands (not owner) and `SetLook` (during a match).

## Observation additions (serialized `core::Observation`)

- `players[i].has_punch: bool`
- `players[i].has_kick: bool`
- `bombs[i].slide: null | {"dir": "Up"|..., "step_left": u32}`
- `players[i].malus: null | {"kind": "Slow"|"HyperSpeed"|"SmallBomb", "ticks_left": u32}`
- `bombs[i].flight: null | {"dir": "Up"|..., "hop_left": u32}`
- `powerups[i].kind` and event kinds include `Punch`, `Slow`, `HyperSpeed`, `SmallBomb`.
- `config` adds `malus_ticks` and `punch_hop_ticks`.

## Tests that bind this contract

Lenient decoding table (hostile and old looks); colour resolution and notice; `SetLook` refused during a match; looks visible to every client in `Lobby`; `Input` without `punch`; dev commands refused without `--dev` and by non-owners; value validation; unknown messages ignored by an old client; fuzz-style random frames including the new types; `docs/PROTOCOL.md` lists every new variant (existing docs test).
