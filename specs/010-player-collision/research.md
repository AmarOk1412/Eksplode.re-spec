# Phase 0 Research: Players Pass Through Each Other

## 1. Where player-vs-player blocking currently lives

**Decision**: There are exactly two places in the codebase that treat another living player's tile as an
obstacle, and both must change together:

1. `crates/core/src/step.rs`, `move_player`'s `blocked` expression:
   ```rust
   let blocked = state.arena.tile(to) != Tile::Floor
       || state.bombs.iter().any(|b| b.pos == to && b.flight.is_none())
       || state.players.iter().any(|o| o.alive && o.pos == to);
   ```
   The third disjunct is the one to remove.
2. `crates/bots/src/pathing.rs`, `passable()`:
   ```rust
   pub fn passable(obs: &Observation, p: Pos) -> bool {
       obs.arena.tile(p) == Tile::Floor
           && !obs.bombs.iter().any(|b| b.flight.is_none() && b.pos == p)
           && !obs.players.iter().any(|o| o.alive && o.id != obs.me && o.pos == p)
   }
   ```
   The third conjunct is the mirror of the same rule for bot route planning and must be removed too, or
   bots would keep detouring around players while humans walk straight through them — violating FR-004
   (identical behavior for humans and bots).

**Rationale**: Both functions independently encode "is this tile currently occupied by a living other
player," and both currently answer "yes → blocked." The spec requires the answer to always be "no" for
this specific obstacle type, uniformly.

**Alternatives considered**: Introducing a per-mode or per-config flag to toggle player collision was
considered, since the spec says "This applies to all game modes" which could imply a future configurable
toggle. Rejected: nothing in the spec asks for a toggle, the config surface (`RoomSettings`) has no field
for it, and adding one would be speculative scope the user did not request (YAGNI).

## 2. Why no other call site needs to change

**Decision**: No other obstacle/occupancy check needs to change.

**Rationale**, scanned call sites that reference player position for occupancy-like purposes:

- `crates/core/src/punch.rs::free_square` (used by `kick.rs` to decide whether a kicked/punched bomb can
  slide onto a square) also excludes squares with a living player. This governs whether a *bomb* can
  occupy a player's square, not whether a *player* can occupy another player's square — a different
  relationship the spec does not mention changing. Left as-is.
- `crates/core/src/chaser.rs` (bonus-spawn placement avoiding occupied tiles) and
  `crates/core/src/closing.rs` (arena-shrink elimination check) both test "is a player standing here" for
  unrelated purposes (where to place a pickup; whether to eliminate someone), not for blocking movement.
  Left as-is.
- `kick::start` (called from `move_player` when `blocked` is true) only acts when the obstacle at `to` is
  specifically a landed, non-sliding bomb (`bombs.iter().position(|b| b.pos == to && ...)`); it already
  no-ops for any other kind of obstacle, including a player, so removing player-occupancy from `blocked`
  does not change `kick::start`'s behavior at all — it simply stops being reached for the
  player-occupancy case (the move now just succeeds before `kick::start` would have been called).

**Alternatives considered**: Auditing and changing `free_square` in tandem was considered, since it
"sounds similar." Rejected after confirming it is a bomb-placement/slide rule, not a player-movement
rule — the spec's FR-002 explicitly keeps bomb collision unchanged, and `free_square` is exactly that
bomb-side rule.

## 3. Rendering/representation readiness for overlapping players

**Decision**: No data-model change is needed to represent two players on one tile — `Player`/`Participant`
already store an independent `pos: Pos` per player with no uniqueness constraint anywhere in `core`
(confirmed: nothing asserts or relies on distinct positions outside the now-removed `blocked` checks).
The web renderer already draws each player from its own `pos` independently; verifying it doesn't
glitch when two positions coincide is a smoke-test item (quickstart.md), not a code-design item, since
no invariant currently *prevents* two players sharing a tile in the rare cases where it can already
happen today (e.g., simultaneous elimination-adjacent edge cases, or future features) — this change just
makes it a routine, common occurrence instead of a rare one.

**Rationale**: Keeps the fix minimal, matching the spec's explicit assumption that only movement blocking
changes, not representation or rendering.

**Alternatives considered**: Adding a "stack order" or z-index field to `Player` to make overlap rendering
deterministic was considered. Rejected as speculative — the spec's Assumptions explicitly scope visual
layering as "no visual change beyond what is needed to render overlapping players without crashing or
glitching," which the existing independent-position rendering already satisfies; add this later only if
the quickstart smoke test finds an actual visual defect.
