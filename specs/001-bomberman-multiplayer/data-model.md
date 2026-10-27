# Data Model: Bomberman Multiplayer Game

Types live in `crates/core` unless noted. All are `Clone`, `Debug`, `PartialEq`, and (for network
types) `Serialize`/`Deserialize`.

## Constants (defaults, in `Config`)

| Name | Value | Notes |
|------|-------|-------|
| `width` x `height` | 15 x 13 | odd sizes; border walls |
| `tick_hz` | 20 | |
| `fuse_ticks` | 60 | 3 s |
| `flame_ticks` | 10 | 0.5 s |
| `round_limit_ticks` | 3600 | 3 min, then draw (FR-007) |
| `base_move_cooldown` | 4 | ticks per move |
| `min_move_cooldown` | 2 | after speed power-ups |
| `start_bombs` / `start_range` | 1 / 2 | |
| `max_bombs` / `max_range` | 6 / 8 | caps |
| `block_density` | 0.7 | of eligible tiles |
| `powerup_chance` | 0.3 | per destroyed block |

## Entities

### Arena (spec: Arena)
- `tiles: Vec<Tile>` row-major, `Tile = Floor | Wall | Block`
- Fixed walls on the border and at every (even x, even y) tile.
- Corner start areas: start tile + 2 adjacent tiles always `Floor`.
- Generated from `(Config, seed)`.

### Participant (spec: Participant)
- `id: PlayerId(0..4)`, `color: Color` (Red, Blue, Green, Yellow by id)
- `pos: (u8, u8)`, `alive: bool`, `facing: Dir`
- `bombs_max`, `bombs_active`, `range`, `move_cooldown`, `cooldown_left`
- `controller: Controller` (Local, Remote, Bot, Empty) - kept in the lobby/session layer, not in `core` rules

### Bomb (spec: Bomb)
- `owner: PlayerId`, `pos`, `ticks_left`, `range`, `owner_left_tile: bool` (owner may walk off it until false -> blocks)

### Flame / Blast (spec: Blast)
- `pos`, `ticks_left`; created when a bomb detonates (cross shape, stops at walls, destroys first block, passes through nothing else).

### PowerUp (spec: Power-up)
- `pos`, `kind: ExtraBomb | Range | Speed`; destroyed by flames; collected by walking onto the tile.
- Hidden under `Block` tiles via `hidden_powerups: Map<pos, kind>` decided at generation (deterministic).

### Action / PlayerInput (spec: Action)
- `PlayerInput { mv: Option<Dir>, bomb: bool }`; `Dir = Up | Down | Left | Right`. "Do nothing" is the default.

### GameState (spec: Round)
- `tick: u32`, `arena`, `players: [Participant; 4]` (unused slots `alive = false` and spawn nothing), `bombs`, `flames`, `powerups`, `rng`, `phase: Playing | Ended(Outcome)`
- `Outcome = Winner(PlayerId) | Draw`

### Event
`Moved`, `BombPlaced`, `BombExploded`, `BlockDestroyed`, `PowerUpSpawned`, `PowerUpCollected`, `PlayerEliminated`, `RoundEnded(Outcome)`. Used by frontends for effects and by tests for assertions.

### Observation (bots, clients)
Read-only view of `GameState` without the RNG and without hidden power-ups under blocks: arena, players, bombs, flames, visible power-ups, own id, tick.

### Lobby / Match (spec: Lobby, Match)
- `Slot { controller: Controller, name: String, ready: bool }` x 4
- `MatchState: Lobby | Playing | Results`
- Defined in `crates/net` (`lobby.rs`) and reused by `cli`/`gui` for local matches.

## Rules and validation (map to FRs)

| Rule | FR |
|------|----|
| Move only to a `Floor` tile with no bomb and no living player; cooldown must be 0 | FR-002 |
| Place bomb if tile has none, `bombs_active < bombs_max`, player alive | FR-003 |
| Detonation produces flames in 4 directions up to `range`; walls stop; first block destroyed and stops the ray | FR-003 |
| Flame touching a bomb sets its `ticks_left = 0` in the same tick (iterative until stable) | FR-004 |
| Alive player on a flame tile is eliminated | FR-005 |
| Destroyed block may reveal a power-up; flame destroys visible power-ups | FR-006 |
| Round ends when `alive <= 1` (Winner / Draw) or tick >= limit (Draw) | FR-007 |
| Inputs for dead or missing slots are ignored; invalid values rejected at the net layer | FR-015 |
| Same `(Config, seed, input log)` gives identical state | FR-017 |

## State transitions (Round)

```text
Lobby -> Playing --(alive <= 1 or tick limit)--> Results -> Lobby (rematch) | exit
```

Within a tick (`step`) order: apply moves (rotating order) -> place bombs -> collect power-ups ->
tick fuses -> detonate (with chain reactions) -> apply flames (eliminations, block destruction) ->
tick flames -> check end of round.
