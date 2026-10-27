# Data Model: Game Modes and Room Settings

All additions are backward compatible: new `Config` and observation fields carry `#[serde(default)]`; the defaults reproduce the behaviour of feature 003 except for the end pattern and the removal of the time draw in Normal (FR-014).

## RoomSettings (core `room.rs`)

| Field | Type | Default | Rule |
|-------|------|---------|------|
| `name` | `String` | "" (host fills "<owner>'s room") | trimmed, at most 24 chars, no control characters |
| `mode` | `Mode` | `Normal` | `Normal`, `Team`, `Chaser` |
| `size` | `BoardSize` | `Small` | `Small` 15x13, `Medium` 19x15, `Large` 25x19 |
| `minutes` | `u8` | 3 | one of 2, 3, 5, 10 |
| `friendly_fire` | `bool` | `true` | used by Team only; shown only for Team |

Methods: `sanitize(self) -> RoomSettings` (applies the limits and defaults), `to_config(&self, base: &Config) -> Config`, `participants_needed(&self) -> (min, max)` (Normal 1..4 including bots, Team exactly 4, Chaser exactly 4) and `label()` for display. Wire form: JSON object, lenient decoding (`net/src/room.rs`).

`Mode` strings on the wire: `"normal"`, `"team"`, `"chaser"`; sizes `"small"`, `"medium"`, `"large"`.

## Config additions

| Field | Default | Meaning |
|-------|---------|---------|
| `mode: Mode` | `Normal` | selects rules and arena layout |
| `friendly_fire: bool` | `true` | Team only |
| `closing_ticks: u32` | `60 * tick_hz` | duration of the pattern (Normal, Team) |
| `closing_lead_ticks: u32` | 20 | countdown of an announced square |
| `chaser_bombs: u8` | 2 | bombs of the chaser |
| `chaser_hearts: u8` | 3 | hearts of the chaser |
| `shield_ticks: u32` | 40 | protection after a heart is lost |
| `chaser_pillar_mod: u8` | 3 | keeps pillars with `(x/2 + y/2) % mod == 0` |
| `chaser_block_density: f32` | 0.12 | blocks in the Chaser layout |
| `bonus_every_ticks: u32` | 100 | Chaser bonus rhythm |
| `bonus_cap: u8` | 6 | most bonuses on the board at once |

`validate()` rejects `closing_lead_ticks == 0`, `chaser_hearts == 0`, `chaser_pillar_mod == 0`, `bonus_every_ticks == 0`, `chaser_bombs == 0` and densities outside 0..1. `closing_ticks == 0`, or not shorter than `round_limit_ticks`, turns the end pattern off (the limit then ends the round as a draw), which keeps tools and tests with short rounds working; room settings always give at least 2 minutes.

## Participant additions

| Field | Type | Meaning |
|-------|------|---------|
| `team` | `Option<u8>` | 0 or 1 in Team mode, else `None` |
| `role` | `Role` | `Normal` (default), `Chaser`, `Runner` |
| `hearts` | `u8` | 1 for everybody except the chaser (`chaser_hearts`) |
| `shield_left` | `u32` | ticks of protection (chaser after a hit) |

## Flame addition

`Flame.owners: u8`, a bit mask of the slots whose bombs lit the tile (bit `i` for slot `i`; `serde` default 0); a tile lit by a teammate and an enemy keeps both, so friendly fire stays exact. `Flame::owned_by(player)` tests a bit.

## GameState additions

| Field | Meaning |
|-------|---------|
| `closing: Closing` | `{ order: Vec<Pos>, next: usize, elapsed: u32, falling: Vec<Falling> }`; empty and inactive outside Normal/Team |
| `bonus_in: u32` | countdown to the next Chaser bonus |

`Falling { pos: Pos, ticks_left: u32 }`.

`closing.order` is computed once when the pattern starts, from the arena at that tick, by the spiral rule of research R6, so it is a pure function of the state.

## Outcome and Event additions

| Item | Variants |
|------|----------|
| `Outcome` | `Winner(PlayerId)`, `Draw`, + `TeamWin(u8)`, `ChaserWins`, `RunnersWin` |
| `Event` | + `BlockWarned { pos }`, `BlockLanded { pos }`, `HeartLost { player, hearts_left }`, `RoleAssigned { player, role }` (round start), `PatternStarted` |

## Observation additions

`Observation` gains `falling: Vec<Falling>` and the extra participant fields above (all public information). `config` already carries `mode` and `friendly_fire`.

## Lobby and host additions

| Item | Meaning |
|------|---------|
| `Lobby.settings` | the current `RoomSettings` (serde default) |
| `Lobby::can_start(&settings)` | Normal: at least 1 participant as today; Team and Chaser: exactly 4 participants (humans and bots) |
| host `room_open` | true while any human is connected |
| host `defaults: RoomSettings` | start-up defaults (server options) |

`AddBot`/`RemoveBot` keep their rules; a start with the wrong number is refused with `CannotStart` and a message that says how many are missing and that bots can fill the slots (the owner adds them).

## Validation summary

| Input | Rule | Failure |
|-------|------|---------|
| `RoomSettings` from the network | lenient decode then `sanitize` | never fails |
| `SetRoom` | owner, lobby phase | `NotOwner`, `WrongPhase` |
| `Start` | `can_start` for the mode | `CannotStart` |
| `Join.room` | only when the room is created by that join | ignored otherwise |

## State transitions

```text
room closed --first Join (with or without room)--> lobby (owner = joiner)
lobby --SetRoom (owner)--> lobby (settings changed)
lobby --Start (owner, can_start)--> match (Config = settings.to_config(base); roles and teams assigned)
match --outcome--> results --Rematch / back--> lobby (settings kept)
lobby --last human leaves--> room closed (settings forgotten)
```
