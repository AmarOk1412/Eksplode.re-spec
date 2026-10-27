# Contract: Powers, Maluses, Punch and Drops (rules in `core`)

## Step order

Within one tick (additions in bold): cooldown timers, flame timers, fuse timers, **malus timers (end -> `MalusEnded`)**, moves (rotating priority; pickups on arrival; **a move into a landed bomb by a player with the kick starts a slide**), **bomb flights (hops and landings)**, **bomb slides (steps and stops)**, **punches**, bomb placement, detonation with chain reactions (**drops decided for destroyed blocks**), eliminations (**clear malus**), end-of-round check.

## Pickups

| Kind | On pickup |
|------|-----------|
| `ExtraBomb` | `bombs_max = min(bombs_max + 1, max_bombs)` |
| `Range` | `range = min(range + 1, max_range)` |
| `Speed` | `move_cooldown = max(move_cooldown - 1, min_move_cooldown)` |
| `Punch` | `has_punch = true` |
| `Kick` | `has_kick = true` |
| `Slow`, `HyperSpeed`, `SmallBomb` | `malus = { kind, ticks_left: malus_ticks }` replacing any current malus; `MalusStarted` |

Every pickup removes the item and emits `PowerUpCollected`.

## Maluses

- Effective move cooldown: none `move_cooldown`; `Slow` `ceil(move_cooldown * 4 / 3)`; `HyperSpeed` `max(1, move_cooldown / 2)`.
- `SmallBomb`: a bomb placed while it is active has `range = 1`; bombs placed earlier keep their range.
- At most one malus; a new malus replaces the old; the same malus again restarts the timer; `ticks_left` reaching 0 ends it (`MalusEnded`) in the tick it hits 0; elimination clears it; a new round starts clean.

## Punch

Preconditions: the player is alive, `has_punch`, `input.punch`, and the neighbour square in `facing` holds a landed bomb (not in flight).

1. The bomb gets `flight { dir = facing, hop_left = punch_hop_ticks }` (`BombPunched`).
2. Each tick `hop_left` decreases; at 0 the bomb moves one square in `dir` (`BombHopped`) and `hop_left` resets. A bomb in flight does not block movement or placement, and flames do not trigger it.
3. On reaching a square, it **lands** (`BombLanded`, `flight = None`) when the square has no wall, no block, no living player and no other bomb (landed or in flight). Otherwise it hops on.
4. If the next square is outside the arena, the bomb lands on the nearest free square going back toward its origin (the origin counts as a candidate); if none is free it stays in flight over the origin and tries again on the next hop tick.
5. The fuse keeps counting; a bomb whose fuse reaches 0 in flight explodes at its current square.
6. A punch with no bomb in front, without the power, or while the bomb is already in flight does nothing.

Landing on a burning square or on a power-up square is allowed (the power-up stays on the board).

## Kick

Preconditions: the player is alive, `has_kick`, and their move targets a square that holds a landed bomb (no flight, no slide) whose next square in the move direction is free (floor, no living player, no other bomb).

1. The player does not move; the bomb gets `slide { dir, step_left = slide_ticks }` (`BombKicked`).
2. Each tick `step_left` decreases; at 0 the bomb moves one square in `dir` (`BombSlid`) and `step_left` resets, unless the next square is blocked (block, wall, any other bomb, a living player, the edge): then the slide ends (`BombStopped`) and the bomb rests where it is. A bomb with no free square behind it never starts.
3. A sliding bomb blocks movement and placement like a landed one, is not immune to flames, keeps its fuse and explodes where it is; chain reactions as usual.
4. A punch on an adjacent sliding bomb ends the slide and starts the flight; a bomb in flight cannot be kicked. Players without the kick treat a bomb as before (blocked).
5. A player stepping onto the next square of a sliding bomb stops it before them.

Bots never kick on purpose but may push by accident only if they hold the kick (they never pick it as a goal differently from other bonuses).

## Drops

- `DropTable` holds eight chances in `PowerUpKind::ALL` order; defaults = `powerup_chance` x shares (`ExtraBomb` .19, `Range` .22, `Speed` .22, `Punch` .03, `Slow` .10, `HyperSpeed` .08, `SmallBomb` .11, `Kick` .05).
- For every block destroyed in a tick (in position order) one value is drawn from the state RNG; at most one item results; sums above 1.0 are scaled by `1 / sum`.
- A chance of 1.0 for one kind and 0.0 for the rest makes every destroyed block drop that kind; all zeros drops nothing.

## Observation

Everybody sees `has_punch`, `malus` and bomb `flight`; nothing about undiscovered items (there is none to leak, items are decided on destruction).

## Bots

Danger map: a bomb in flight counts as exploding at its landing prediction and at its current square. Target scoring: bonuses positive, maluses negative. Bots never punch in this feature.

## Tests that bind this contract

Per-kind pickup; caps; slow and hyper cooldown values and timing (10 s, 200 ticks); small bomb range and earlier bombs unchanged; replacement and restart; elimination and round end clear; punch cases (free next square, block, wall, player, other bomb, player stepping in during flight, edge fallback, no power, no bomb, already flying, fuse ends in flight, chain reaction after landing); drop table (100%, 0%, scaling, defaults share over 1000 and 200 draws, determinism per seed); determinism of whole matches with the new rules.
