# Data Model: Customization, Powers, Sound and Developer Mode

Types are described by what they hold and which rules apply, not by code.

## Look (crate `net`, module `look`)

| Field | Values | Default for slot n |
|-------|--------|--------------------|
| `colour` | `Blue, Black, Red, Purple, Green, Orange, Yellow, Pink, Teal, White` (10) | slot 0 Blue, 1 Black, 2 Red, 3 Purple |
| `gender` | `Male, Female` | slot 0 Male, 1 Female, 2 Male, 3 Female |
| `head` | `None, Cap, TopHat, Crown, Headphones` | `None` |
| `face` | `None, Glasses, Sunglasses, Moustache, Mask` | `None` |

Rules:
- Wire form: JSON object `{"colour":"Blue","gender":"Male","head":"Cap","face":"None"}`; every field is optional on input.
- `Look::from_wire(value, slot)`: any missing, wrongly typed or unknown field becomes the default of that field for `slot`; a non-object becomes the full default. Never fails.
- **Colour uniqueness** among the seated participants: when assigning, if the colour is used by another seated slot, take the next free colour in palette order (wrapping). The assignment reports whether it adjusted.
- Equality is field-wise; two looks differing only in accessories are different looks.

Relationships: one look per slot (`looks: [Look; 4]`), published with the lobby. A slot's look is cleared when the slot becomes `Empty` and set to the slot default when a bot or a terminal player takes it.

State: *default* -> *chosen* (Join) -> *changed* (SetLook, lobby only) -> *locked* (match running) -> back to *changed* when the lobby returns.

## Power kinds (crate `core`)

| Kind | Display name | Class | Effect | Duration | Rarity (default chance per destroyed block) |
|------|--------------|-------|--------|----------|--------------------------------------------|
| `ExtraBomb` | extra bomb | bonus | +1 simultaneous bomb, capped by `max_bombs` | round | 0.060 |
| `Range` | flame | bonus | +1 explosion length, capped by `max_range` | round | 0.072 |
| `Speed` | roller | bonus | move cooldown -1, floor `min_move_cooldown` | round | 0.072 |
| `Punch` | punch | bonus | may punch the bomb on the faced square | round | 0.009 |
| `Kick` | kick | bonus | walking into a landed bomb makes it slide until the next square is blocked | round | 0.015 |
| `Slow` | slow | malus | effective cooldown `ceil(base * 4 / 3)` | `malus_ticks` | 0.030 |
| `HyperSpeed` | hyper speed | malus | effective cooldown `max(1, base / 2)` | `malus_ticks` | 0.024 |
| `SmallBomb` | small bomb | malus | bombs placed get range 1 | `malus_ticks` | 0.033 |

(Defaults = `powerup_chance` 0.3 times shares .19, .22, .22, .03, .10, .08, .11, .05 for `ExtraBomb, Range, Speed, Punch, Slow, HyperSpeed, SmallBomb, Kick`; they sum to 0.3, punch is 3% of drops. The table above lists the kinds by effect; `PowerUpKind::ALL` ends with `Kick` so that the existing indexes do not move.) `PowerUpKind::ALL` has this order and indexes the drop table.

## Participant additions

- `has_punch: bool` - false at round start.
- `has_kick: bool` - false at round start.
- `facing: Dir` (already present) is what the renderer uses for the sprite view.
- `malus: Option<ActiveMalus>` - `ActiveMalus { kind: Slow | HyperSpeed | SmallBomb, ticks_left }`, at most one.
- Derived (not stored): `effective_cooldown(config)`; `placed_range(config)` = 1 with `SmallBomb`, else `range`.

Transitions: pickup of a malus sets/replaces the active malus (timer = `malus_ticks`); each tick decrements; at zero `MalusEnded`; elimination clears it. Picking the same malus again restarts the timer.

## Bomb additions and flight

- `flight: Option<Flight>` where `Flight { dir, hop_left }`: `hop_left` counts ticks until the next hop (starts at `punch_hop_ticks`).
- A bomb with a flight: does not block movement or placement, is not triggered by flames, keeps its fuse; `pos` is the square it is over.

State: *landed* -> (punch) -> *in flight* -> (free square reached) -> *landed*. A bomb whose fuse ends in flight explodes at `pos`.

## Events added

`BombPunched { player, from, dir }`, `BombHopped { pos }`, `BombLanded { pos }`, `MalusStarted { player, kind }`, `MalusEnded { player, kind }`. `PowerUpSpawned`, `PowerUpCollected`, `PowerUpDestroyed` carry the larger `PowerUpKind`.

## Config additions (serde defaults keep old hosts compatible)

`malus_ticks` (10 s x `tick_hz`), `punch_hop_ticks` (2). Validation: both at least 1.

## Drop table (crate `core`, module `drops`)

- Seven chances in `PowerUpKind::ALL` order, each `0.0..=1.0`, finite.
- `DropTable::defaults(powerup_chance)`; `validate()`; `roll(rng)`: one RNG draw; walk cumulative chances; if their sum exceeds 1.0, scale each by `1 / sum`; returns at most one kind.
- Lives in `GameState.drops`; changed between ticks by `set_drops` (host, developer mode) or reset to `defaults`.
- `hidden_powerups` is removed; items appear when a block is destroyed.

## PlayerInput addition

`punch: bool` (default false); with `mv` and `bomb` it forms the input of a tick. The page and terminal send it while the key is down; the host passes it to `step`.

## Host state (crate `net`)

| Item | Meaning |
|------|---------|
| `looks: [Look; 4]` | current look of each slot |
| `dev: bool` | developer mode enabled at start-up |
| `bots_paused: bool` | bots give empty inputs while true (dev only) |
| `drops: DropTable` | current table, applied to each new `GameState` and live (dev only) |

## Audio settings (browser, per device)

`{ music: 0..1, effects: 0..1, muted: bool }`, defaults `0.35 / 0.7 / false`, stored as `eksplode.re.audio.v1`. Invalid or missing values fall back to defaults.

## Stored look (browser, per device)

The last chosen `Look` as the wire object, stored as `eksplode.re.look.v1`.

## Developer state (as shown to clients)

`DevState { paused: bool, drops: [f32; 8] }`, broadcast on change and sent on join when `dev` is on.
