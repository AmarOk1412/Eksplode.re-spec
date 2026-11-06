# Data Model: Revenge Karts

**Status**: Proposed design, not implemented | **Spec**: [spec.md](spec.md)

## Existing State Kept Intact

`GameState` remains the authoritative round owner. `Participant.alive` still means an on-board survivor; `pos` remains their last/current board position and is not overloaded with track coordinates. Existing team, role, owner bits, outcome types, and cumulative statistics retain their meaning. `Observation` continues to copy the authoritative participant, bomb, and flame state.

## Participant Additions

| Field | Type / Default | Meaning and Validation |
|-------|----------------|------------------------|
| `kart` | `Option<KartState>` / `None` | Present only for an active, eliminated Normal/Team participant in a playing round before closing. Incompatible with `alive = true`. |
| `immobilized_left` | `u32` / `0` | Number of future movement passes blocked by direct-hit stun; at most `config.tick_hz`. Zero for karts and permanently eliminated participants. |

Both fields use serde defaults and may omit neutral values when serialized. Old snapshots produce the ordinary legacy state. Powers remain in the existing fields; clearing/resetting them never resets `kills` or `bombs_placed`.

### KartState

| Field | Type | Meaning and Validation |
|-------|------|------------------------|
| `index` | `u16` | Clockwise virtual perimeter index, strictly less than `2 * (width + height) - 4`, computed with widened arithmetic. |
| `entered_tick` | `u32` | Round tick of the elimination creating this kart. Unique per slot per stint because a slot can enter a kart at most once in one tick. Never reused within a round. |
| `cooldown_left` | `u32` | Kart movement cooldown; bounded by `config.base_move_cooldown`, initialized to zero. |
| `travel` | `Option<Dir>` | Only `Left`, `Right`, or `None`, representing current requested perimeter travel for presentation; not inward throw aim. |
| `pending_return` | `Option<Pos>` | Preferred victim square for one earned return, or none. It must be an in-bounds interior position. It is an entitlement, not a reserved square. |

The kart participant's owner/team is inherited from its containing `Participant`; do not duplicate either. No separately persisted vehicle ID, direction-of-aim field, or active-kart-bomb counter is necessary. Helpers derive aim and count outstanding current-stint bombs from authoritative state.

## Bomb Addition

| Field | Type / Default | Meaning and Validation |
|-------|----------------|------------------------|
| `revenge_stint` | `Option<u32>` / `None` | Entry tick of the owner's kart stint that launched this bomb. Ordinary bombs have none. Source owner remains `Bomb.owner`. |

This field survives landing, kick, and subsequent punches. It cannot change bomb ownership or transfer to a different bomb in a chain reaction. A legacy bomb with no provenance cannot earn a return. A source from an earlier stint remains legal round state, but cannot revive a new kart. The entry tick cannot be later than the observation tick.

### Flight Reuse

Keep `Flight { dir, hop_left, origin }` and `Bomb.flight`. At launch set `pos` and `origin` to square one and start the normal hop countdown; the next hop is square two. Check launch feasibility from square two inward, followed by backward fallback through square one. The initial square-one position is airborne and neither blocks nor hits a survivor there. Later flight hops and fuse expiry use the existing engine.

## Flame Addition

| Field | Type / Default | Meaning and Validation |
|-------|----------------|------------------------|
| `revenge_stints` | `[Option<u32>; 4]` / all `None` | For each owner slot, the newest kart entry tick whose own blast contributed to this flame square. |

Keep `owners: u8` unchanged for friendly fire and cumulative kill counters. Detonation accumulates provenance from each exploding bomb's own blast tiles; overlapping flames merge owner bits with OR and per-owner optional stint values with maximum. A revenge value must have its corresponding owner bit set and must not be in the future. New clients use an all-none default for legacy flames.

Why maximum is sufficient: a slot's current kart stint is its newest entry tick; earlier epochs cannot become eligible again. At most four values per tile suffice even if several old throws overlap. An ordinary later blast must not erase a still-active current-stint source, matching the existing merged-flame owner lifetime. Sources expire with their flame, including normal refresh behavior.

## Transient Tick Resolution

Use a bounded, tick-local collection of elimination records with victim slot, position, hurting owner bits, and qualifying revenge sources. Capture owners eligible at tick entry before mutating participation states. Do not serialize these records or retain a kill log. This avoids dependence on mutable participant iteration order.

For each victim, normal kills update exactly as before. Revenge requires an opponent relationship and a source matching an owner's kart stint at tick entry, with that kart still eligible before closing. For multiple victims of one owner, prefer the lowest victim slot. A participant eliminated from the board this tick cannot earn a return from old bombs merely because they now have a kart.

## State Transitions

| From | Trigger | To / Effects |
|------|---------|--------------|
| Living | Eligible elimination; round continues | `alive = false`, new kart at nearest perimeter, clear malus/stun and old powers; stamp elimination order. |
| Living | Elimination after cutoff, Chaser elimination, or final result | Eliminated without kart. Existing outcome rules apply. |
| Kart | Left/right request and cooldown ready | Neighboring perimeter index modulo track length; set base move cooldown. No request stops travel. |
| Kart | Bomb action and current-stint capacity free; feasible line | One provenance-bearing flight; count in total owned outstanding bombs and cumulative bombs placed. |
| Kart | Qualifying opponent kill | Set one pending return, preserving the first earned preferred victim square. |
| Kart with pending return | Safe free square exists | Remove kart, set `alive = true`, restore round-start powers/capacity/range/movement pace, clear elimination order and temporary effects. |
| Kart with pending return | No safe free square; round continues | Remain in kart; no stacked entitlement. Retry after eliminations on the next tick. |
| Kart | Closing begins or round ends | Remove kart and pending return; no further actions this round. |
| Living, including returned | Direct eligible flight hit | Refresh stun to `tick_hz`; no damage, power reset, or action disable beyond movement. |
| Any active participation | Human replaced by bot | Same participant state, owner, team, bombs, timer, kart, and pending return; only controller changes. |
| Any | New round | Ordinary constructor/reset state, all added fields neutral, previous bombs/flames gone. |

## Invariants and Accounting

- `alive` and `kart.is_some()` are mutually exclusive. Inactive slots never gain a kart. Only alive participants contribute to outcomes, player collision, item collection, or return-square occupancy.
- Kart capacity is one outstanding bomb with matching owner and `revenge_stint == kart.entered_tick`, regardless of flight/landing. Earlier ordinary/stale-stint bombs do not consume this one allowance.
- `bombs_active` always counts actual outstanding owned bombs across all origins. A return does not zero it; normal capacity blocks new ordinary placements while that count meets/exceeds reset `bombs_max`.
- One return per stint is implicit: consuming a return removes that kart, and a later death creates a different `entered_tick`. No once-per-round boolean is introduced.
- Safe return square: interior floor, no bomb (airborne or landed), no alive player, no active flame. Items do not block returning. Walls/blocks and remaining survivors are never removed for a return.
- Return priority is `(owner slot)`; candidate priority is `(Manhattan distance to preferred victim position, y, x)`. Reserve each selected square immediately by setting that owner alive before choosing the next one.
- A return clears stun, malus, punch, kick, shield, and prior elimination order; restores Normal/Team round-start hearts and cooldown state; retains team, identity, match statistics, and outstanding bomb ownership.
- Stun is checked during movement; decrement after movement but before flight advance. Fresh post-movement hits set the full `tick_hz`, blocking exactly that many subsequent movement passes. It neither replaces nor pauses a malus.
- Closing due is tested at tick entry, short-circuiting `closing::enabled` before computing the start tick. Existing closing advancement remains after detonation. Match end also clears stun on surviving participants.
- Avoid relying on `Pos`'s derived ordering for return ties: its field order is x then y, whereas this specification requires y then x.

## Validation Boundary

Authoritative state is constructed and mutated only in core; clients cannot submit kart fields or revive themselves. Snapshot ingestion used by renderers should reject malformed typed fields and safely treat contradictory/out-of-range added state as inert instead of indexing unchecked or panicking. Tests cover invalid indices, future source ticks, contradictory alive/kart state, bad travel directions, and malformed provenance arrays, including legacy neutral defaults.