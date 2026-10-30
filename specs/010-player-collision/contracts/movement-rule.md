# Contract: Player Movement Passability Rule

This feature has no wire/API surface change (no new message, no changed message shape in
`crates/net/src/protocol.rs`). The "contract" here is the shared behavioral rule both the core simulation
and bot pathfinding must implement identically.

## Rule

A move from tile `from` to adjacent tile `to` by living player `i` is **blocked** if and only if:

```text
blocked(to) :=
       arena.tile(to) != Floor
    OR exists a landed (non-flying) bomb at `to`
```

Explicitly **not** part of `blocked`:

```text
    (removed) OR exists a living player other than `i` at `to`
```

## Conformance

| Call site | File | Must implement |
| --- | --- | --- |
| Core simulation tick | `crates/core/src/step.rs::move_player` | `blocked` exactly as above |
| Bot route planning | `crates/bots/src/pathing.rs::passable` | the positive form (`passable(p) := !blocked(p)`) exactly as above, for every candidate tile `p` other than where a bot already is |

Both call sites MUST agree at all times — a player must never be able to walk somewhere a bot's planner
considers impassable for the same reason, or vice versa, since Principle V requires bots and humans to
share one movement interface.

## Non-goals (explicitly unaffected by this contract)

- Bomb placement, blast propagation, and blast-vs-player damage resolution (`bomb.rs`, `eliminate()` in
  `step.rs`) — already per-player, independent of tile sharing.
- Bomb kicking/sliding target validity (`punch::free_square`, used by `kick.rs`) — a bomb's ability to
  occupy or slide onto a player's tile is unchanged.
- Power-up pickup (`move_player`'s post-move powerup check) — already per-player and per-tile, unaffected
  by whether another player also stands there.
- Any mode-specific "catch"/"tag" rule — none currently exists in `core`; if added later it is a separate
  feature, not implied by this contract.
