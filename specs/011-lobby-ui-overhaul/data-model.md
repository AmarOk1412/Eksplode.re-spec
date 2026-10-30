# Phase 1 Data Model: Menu, Lobby, Customization and Scoreboard Reorganization

All entities below are additive to existing types; no existing field is removed from wire-stable
structs except `Look.gender`, which is downgraded to "read but ignored" for backward compatibility (see
research.md §5) rather than hard-removed from the wire decoder.

## Room Configuration (existing `core::RoomSettings`, unchanged shape)

| Field | Type | Notes |
| --- | --- | --- |
| `name` | `String` | ≤16 chars, empty = default name |
| `mode` | `Mode` | Normal / Team / Chaser (existing) |
| `size` | `BoardSize` | existing |
| `minutes` | `u8` | one of the existing `MINUTES` presets |
| `friendly_fire` | `bool` | Team mode only |

No new fields. FR-011 is satisfied by moving *where* these are edited (a configuration panel) not by
changing what they are; `password` is carried alongside `RoomSettings` in the existing v2 `SetRoom`
request (`password: Option<Secret>`), not inside `RoomSettings` itself.

## Player Slot (existing `core::lobby::Slot`, plus one new core method)

| Field | Type | Notes |
| --- | --- | --- |
| `controller` | `Controller` (`Local`/`Remote`/`Bot`/`Empty`) | existing |
| `name` | `String` | existing |
| `ready` | `bool` | existing |

**New method**: `Lobby::remove_bot_at(&mut self, slot: usize) -> Result<(), LobbyError>` — clears a
specific bot slot (errors with `LobbyError::SlotOutOfRange` or a new `LobbyError::NotABot(usize)` if the
slot isn't a bot), replacing the current slot-less `remove_bot` (which picks `last_bot()` for the caller).
**New method**: `Lobby::fill_vacant_with_bots(&mut self)` — calls `add_bot()` for every remaining empty
slot (used by the bot-fill-on-vacancy policy in research.md §3); idempotent, no-op if no empty slot
remains.

## Character Appearance (`net::look::Look`, `web-render::look::Look`)

| Field | Type | Notes |
| --- | --- | --- |
| `colour` | `Colour` (10 values) | existing, unchanged |
| `head` | `Head` (5 values) | existing, unchanged |
| `face` | `Face` (5 values) | existing, unchanged |
| `costume` | `Costume` (new enum) | new player-facing category; default `Costume::None` |
| ~~`gender`~~ | `Gender` | **removed from the wire-level, player-settable `Look`**; rendering keeps an internal, non-serialized, slot-derived gender detail per research.md §5 (not part of this `Look` struct's public/wire surface) |

`Look::from_wire` (lenient decode) continues to accept and discard a legacy `gender` key from old
clients/stored state instead of erroring, matching the crate's existing leniency doctrine.

## Match Statistics (new fields on `core::state::Player`)

| Field | Type | Notes |
| --- | --- | --- |
| `kills` | `u32` | incremented per research.md §4; 0 at match start |
| `bombs_placed` | `u32` | incremented on every successful bomb placement by this player, regardless of outcome; 0 at match start |
| `eliminated_order` | `Option<u32>` | `None` while alive/surviving to the end; set once, to the next unused 1-based ordinal, the tick a player is eliminated |

These three fields are per-match (reset when a new match starts from the lobby) and are carried in the
existing per-tick `Observation`/`Snapshot` types (`crates/core`, relayed unchanged through
`crates/net/src/protocol.rs`) so the web client can render the scoreboard without a separate query.

## Match Scoreboard Entry (derived, not a new wire type — computed client-side from the fields above)

| Field | Source |
| --- | --- |
| `name` | `Lobby::slots[i].name` |
| `kills` | `Player.kills` |
| `bombs_placed` | `Player.bombs_placed` |
| sort key | `eliminated_order` (None first, i.e. last-alive/winner first), then `eliminated_order` descending among eliminated players, then `kills` descending, then slot index ascending — see research.md §4 |

## State Transitions

- **Slot vacancy → bot fill**: `Empty` slot (room creation, or any non-owner-initiated vacancy while in
  lobby phase) → `Bot` via `fill_vacant_with_bots`. An owner-initiated `RemoveBot`/`RemovePlayer` leaves
  the slot `Empty` with a "+" affordance instead (research.md §3).
- **Player eliminated**: `Player.alive: true → false` (existing) now also stamps
  `eliminated_order: None → Some(next_ordinal)` and increments `kills` on each contributing non-victim
  player, at the same two call sites that already flip `alive`.
- **Bomb placed**: existing bomb-creation path now also increments the placing player's `bombs_placed`.
- **Look edited**: `SetLook` with a `gender` key present is accepted but the key is ignored (no error,
  matching existing lenient decoding); `costume` key is read/validated like `head`/`face` are today.
