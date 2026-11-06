# Gameplay Contract: Revenge Karts

**Status**: Proposed core contract | **Related**: [data model](../data-model.md), [specification](../spec.md)

## Ownership

All decisions in this document belong to `eksplode-re-core`. Bots and renderers may call pure geometry/target-prediction helpers, but cannot independently decide damage, attribution, return eligibility, or board mutation. Proposed public helper names may follow existing naming conventions during implementation; their behavior below is the contract.

## Perimeter Geometry

For arena width `w` and height `h`, define a virtual ring of length `L = 2 * (w + h) - 4`. Compute with `u16` or wider. Ring coordinates describe geometry, never arena occupancy:

| Segment | Count | Clockwise Virtual Coordinates | Inward Aim |
|---------|-------|-------------------------------|------------|
| Top | `w - 1` | `(0, 0)` through `(w - 2, 0)` | Down |
| Right | `h - 1` | `(w - 1, 0)` through `(w - 1, h - 2)` | Left |
| Bottom | `w - 1` | `(w - 1, h - 1)` through `(1, h - 1)` | Up |
| Left | `h - 1` | `(0, h - 1)` through `(0, 1)` | Right |

Index zero is top-left; right input increments and left input decrements modulo `L`. Each corner begins its next clockwise edge, as specified. These virtual coordinates are not `Participant.pos` and do not imply driving on wall tiles. Display helpers project the ring onto the exterior gutter, with corners following the outside rectangle.

For entry, choose the perimeter position minimizing Manhattan distance to the elimination square. Break an equal-distance edge choice by Top, Right, Bottom, Left, then lowest offset on that edge. For a lane, clamp the parallel coordinate to the interior interval: top/bottom use `x` in `1..=w-2`, left/right use `y` in `1..=h-2`. Clamping corner lanes avoids targeting a line composed solely of border walls.

The first playable square is `(lane_x, 1)` from Top, `(w-2, lane_y)` from Right, `(lane_x, h-2)` from Bottom, and `(1, lane_y)` from Left. Square two is the next interior coordinate in the inward direction. A fixed wall or destructible block at either coordinate does not change distance counting.

Expose/test pure helpers for ring length, validated index-to-edge projection, nearest entry, one-step travel, inward square one/two, and throw landing prediction. These must agree on every edge/corner for all room sizes and a maximum valid core arena. Geometry errors yield no action/typed error as appropriate, never an unchecked coordinate cast.

## Input Dispatch and Throwing

- A participant alive at tick entry follows existing board controls. A participant with a kart at tick entry follows kart controls. A newly eliminated participant cannot use a second action as a kart during that same tick.
- Cart `mv = Left` means counterclockwise, `Right` means clockwise, and `None`, `Up`, or `Down` means stationary. Punch is ignored. The ordinary bomb action launches; movement before launch chooses the current edge/lane.
- Count only current-stint outstanding kart bombs for the one-bomb kart allowance. Check the line for any free landing candidate before inserting a bomb. A refused attempt changes neither statistics nor capacity.
- Launch at square one in flight with normal `fuse_ticks`, `start_range`, and `punch_hop_ticks`; the first candidate landing/hit is square two. Increase cumulative bombs placed and actual active ownership count on a successful launch.
- The first hop is not advanced again in the launch tick. Subsequent hops test current occupancy, not a frozen prediction. A free square is floor with no living player or other bomb, airborne or landed. Items do not block.
- On a candidate occupied by a survivor, apply one second of movement stun if their team relationship permits; the bomb continues. Walls, blocks, and bombs are skipped without being pushed. No hit occurs at square one merely from initial launch.
- At the far edge, preserve punch's backward fallback through the origin. A fallback is a search, not simulated collision travel; it does not invent hits on intervening players. If a once-feasible line becomes fully occupied, keep the flight pending until it can land or its fuse expires.
- Fuse countdown, airborne explosion, invulnerability to chain triggers while airborne, landing, subsequent kicks/punches, and ordinary blast damage remain owned by existing bomb/punch/kick code. Predictions shared with bots must use the same second-square start and fallback semantics.

## Tick Ordering

Maintain the existing non-kart combat order. The intended sequence is:

1. If not playing, return no events. Capture tick-entry alive flags and eligible kart stints.
2. Compute whether closing is due; if so, remove karts and pending returns and disable their actions before doing anything else. This does not land closing blocks early.
3. Advance existing cooldowns, flames, fuses, shield, and malus timers; advance kart movement cooldowns. Do not decrement stun yet.
4. Process ordinary survivor movement and existing-kart movement in the existing rotating slot priority. Check stun before any survivor move or kick attempt. Kart left/right does not change board-facing fields or collect items.
5. Decrement existing stun counters, then advance existing bomb flights. A kart-origin flight hit now sets a full `tick_hz` stun. Continue existing slides and allowed survivor punches.
6. Dispatch allowed bomb actions: normal placement for living participants and kart launches for existing eligible karts. Do not apply a launch twice through ordinary placement/auto-bomb.
7. Detonate with existing chain rules and provenance merging; run closing advancement and Chaser bonus spawning in their existing positions.
8. Gather and apply all flame eliminations, preserve existing kill statistics, and stamp this tick's elimination order. Closing-block deaths remain ordinary deaths with no revenge source. Record qualifying opponent kills against tick-entry kart stints, independently of victim iteration order.
9. Create karts for newly eliminated eligible participants and assign new entry ticks. These new karts cannot qualify against earlier bomb stints or act this tick.
10. Assign at most one pending return per eligible owner and resolve all pending returns in owner-slot order against the final current board/flames/occupants. Clear the revived owner's elimination order. A previous pending return uses its original preferred position.
11. Increment the tick and apply existing alive-only Normal/Team/Chaser outcomes. If ended, clear all remaining kart/pending/stun state and emit existing `RoundEnded`. Do not defer a winner or draw for an unavailable return.

Closing due requires Normal/Team with closing enabled and `closing.started || tick >= start_tick`. Short-circuit the enabled check to avoid subtracting an invalid disabled interval. It remains due if a late snapshot or manually constructed state skipped the exact start tick.

## Credit and Return

Preserve current owner-bitmask normal kill credit. A chain-triggered bomb contributes its own owner's source only to its own blast, not the triggering owner's source to its entire blast. A qualifying kart participant must have an actual current-stint source on a flame square that kills an opponent, remain eligible before closing, and have been in that kart at tick entry. Direct hits do not grant returns.

For each owner, choose the lowest-slot qualifying victim if they do not already have a pending entitlement. Search safe return squares by `(distance, y, x)` from the preferred victim position and resolve owners by slot. Exclude active flames, all bombs, walls/blocks, and other survivors. A power-up can share a return square but is not collected until an ordinary pickup action/move does so under existing rules.

Restore round-start Normal/Team powers, retain identity/team/statistics and bomb ownership, and clear all temporary effects and elimination order. Retain the actual number of owned outstanding bombs even when above the reset placement capacity. No shield, additional life stock, terrain removal, or opponent displacement is granted.

## Outcome Examples

| Situation | Required Result |
|-----------|-----------------|
| Kart kills one opponent while multiple sides remain | Return immediately if safe; otherwise remain kart with one pending return. |
| Kart kills the last opposing survivor and has a safe return | Resolve return, then determine winner from final living participants. |
| Last opposing survivor dies and no safe return exists | Do not delay the existing winner/draw for the pending return. |
| Both teams' last living members die, with eligible safe returns | Resolve returns by slot, then recompute teams from living participants. |
| A teammate dies to a kart bomb with friendly fire enabled | Preserve ordinary kill counters; no return for that teammate kill. |
| Old ordinary bomb, stale kart bomb, or stale flame earns normal kill credit | No revival unless its provenance matches the owner's current eligible tick-entry kart. |
| Kart kill on the closing-start tick | Existing bomb/explosion resolves, but no kart throw or return is permitted. |
| Returned player reaches closing | Keep them alive on the board; normal closing rules continue. |

## Bot Contract

Dispatch karts before dead-player exits in both `decide` and `plan`. A kart-specific held-direction path must not use `me.pos` board-passability checks; otherwise bots stop at their death square. Keep existing difficulty think interval, reaction-filtered observation, and seeded mistake cadence. Reset cached board navigation when entering/leaving a kart so old paths cannot leak between states.

Use public observation and shared prediction helpers to select an opponent lane, choose shortest ring travel with a fixed clockwise tie, and throw when the visible predicted landing/blast threatens an opponent. Never deliberately target teammates. Legal mistakes may skip a throw, hesitate, or choose the opposite ring direction; they must never emit up/down as kart travel. Returned bots resume normal survivor planning; their safety logic must respect stun as unavailable movement, without pretending a stunned player can escape before detonation.