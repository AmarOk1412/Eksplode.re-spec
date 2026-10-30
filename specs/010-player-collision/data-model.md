# Phase 1 Data Model: Players Pass Through Each Other

No new entities, fields, or state transitions. This feature removes an invariant rather than adding one:

## Removed invariant

**Before**: "No two living players may share a tile as a result of movement" (enforced only at the
moment of a move, in `core::step::move_player`, and mirrored as a route-planning constraint in
`bots::pathing::passable`).

**After**: No such invariant exists. `Player.pos` (and `Participant.pos` as seen by bots/observations) may
equal another living player's `pos` for any duration, as a normal and expected state, not an edge case.

## Unchanged entities

- `core::state::Player` / `Participant`: no field changes. `pos: Pos` already had no uniqueness
  constraint in its type; the constraint lived only in the call sites listed in research.md §1.
- `core::state::Bomb`, `Flame`, `Tile`: unchanged — wall/block/bomb blocking (FR-002) and blast
  attribution (FR-003, FR-005) are untouched and already operate per-player regardless of tile sharing
  (e.g., `eliminate()` in `step.rs` already checks `f.pos == p.pos` per player independently, so two
  players on the same burning tile are already each evaluated and eliminated independently — no change
  needed there).
