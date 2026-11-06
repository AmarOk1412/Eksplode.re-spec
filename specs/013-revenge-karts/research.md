# Research: Revenge Karts

**Date**: 2026-11-02 | **Feature**: [spec.md](spec.md)

All decisions below are design proposals, not implemented behavior. Research used the current working tree, including existing unrelated changes, without modifying gameplay code.

## 1. Extend the existing simulation, not a second engine

**Decision**: Add a focused core kart module and extend existing state, bomb, punch, and step owners. Reuse current configuration defaults: 20 Hz, fuse 60 ticks, flame 10 ticks, movement cooldown 4 ticks, hop 2 ticks, start capacity 1 and range 2. Derive stun length from `tick_hz`.

**Rationale**: `game/crates/core/src/config.rs` defines these values and validates positive tick frequency. `step.rs` is the controlling deterministic loop. `PlayerInput` already expresses the needed left/right/bomb actions. Existing engine reuse meets the domain-library guidance without importing a competing rules engine.

**Alternatives considered**: New dependencies, client-side kart simulation, new room settings, or a separate game mode add complexity and violate the scope or core/frontend boundary.

## 2. Preserve alive semantics and use a virtual perimeter

**Decision**: Keep `Participant.alive` true only for ordinary board survivors. Add defaulted optional `kart`, with an index on a virtual clockwise perimeter and a stint identified by its entry tick. Use `u16` index arithmetic; all supported core board dimensions fit. Karts do not have arena occupancy.

**Rationale**: `state.rs`, `step.rs`, `team.rs`, and renderer paths use alive for collisions, damage, outcomes, and display. Reinterpreting it would make karts invulnerable survivors that prolong rounds. `arena.rs` has a wall border; define inward lanes from the first interior square, not from border tiles. Detailed mapping is in the gameplay contract.

**Alternatives considered**: Put carts on wall tiles or rewrite terrain to create a track; both conflate presentation with board collision. An `u8` perimeter index overflows on valid larger core arenas. An entirely new participation enum would unnecessarily break existing snapshots and alive-based consumers.

## 3. Reuse flight while distinguishing kart direct hits

**Decision**: Keep `Bomb.flight` and existing punch-hop timing. Launch the new bomb at the first interior square in flight, so the first hop evaluates square two; no direct hit or landing is evaluated at square one. Add bomb revenge provenance independently of whether it is currently airborne. Only provenance-bearing flights apply kart direct-hit immobilization. Ordinary punched bombs remain unchanged.

**Rationale**: `punch.rs::advance` hops then checks `free_square`; that predicate excludes all other bombs, including airborne ones. The current edge fallback searches backward to flight origin. `bomb.rs::detonate` allows an airborne fuse to expire and ignores airborne bombs when chain-triggering. Preserve those rules and their prediction helpers. The spec's broad phrase "other bombs" therefore includes airborne bombs at candidate landing coordinates.

**Alternatives considered**: Teleport to a precomputed landing position loses dynamic player-hit behavior. A separate complete flight engine duplicates punch semantics. Applying stun to all punched bombs silently expands feature scope.

**Dynamic full-line failure**: Check that some landing square exists before launch, so a fully blocked line consumes nothing. If all landing squares become occupied after launch, retain the in-flight bomb until a later valid hop/fallback or fuse expiry; do not undo elapsed time, impacts, or capacity. This is the existing punch behavior, not a retroactive refusal.

## 4. Preserve attribution and retain stint provenance through flames

**Decision**: Keep existing `Flame.owners` and normal `kills` accounting. Add a bomb's optional kart-stint entry tick and a flame's four optional per-owner latest kart-stint entry ticks. For each blast tile merge owner bits as today and merge provenance by per-owner maximum. Qualify revenge only for an opponent victim, an actual hurting blast contribution on that square, and an owner who was already in the matching kart at tick entry.

**Rationale**: `step.rs::eliminate` credits every other slot present in the hurting flames' owners bitmask. `bomb.rs::detonate` gives each bomb's own blast its own owner, even when another bomb triggers it, and merges overlapping flame ownership. Normal kill counters cannot identify a qualifying kart throw. A bomb-only field is insufficient once the bomb is removed but its flames remain. Entry ticks increase monotonically within the existing round tick lifetime, so a per-owner maximum preserves every still-eligible current stint and safely discards older stints.

**Alternatives considered**: Grant revenge from any counter increment (wrong for old bombs and teammate kills); transfer chain credit to its trigger (changes existing rules); retain an unbounded list of past throws (unnecessary). Returning whenever an owner has ever been in a kart permits stale-flame farming across lives.

## 5. Stun is an independent timer, not a malus

**Decision**: Add `immobilized_left`. Movement tests the current timer, then timers decrement after the movement pass and before flight hits; a hit sets `tick_hz`. This blocks exactly the next `tick_hz` movement opportunities. It does not change move cooldown progression, facing restrictions elsewhere, bomb/punch actions, or malus duration.

**Rationale**: Current `step.rs` moves survivors before advancing flights, so decrementing a fresh hit before the next movement would shorten the effect. Maluses are mutually exclusive powers; replacing slow or auto-bomb with stun would change existing behavior. Hits refresh, never sum, and friendly-fire-disabled teammates are skipped using the existing team relationship.

**Alternatives considered**: Reuse `cooldown_left` (conflates ordinary movement cadence and stun), replace `malus` (loses powers), or reorder all bomb flights before movement (changes existing combat).

## 6. Cutoff early; preserve falling-block and explosion order

**Decision**: At tick entry compute closing due as `closing::enabled(state)` and either already started or `tick >= closing::start_tick(state)`. Purge karts/pending returns and disable kart actions immediately. Keep `closing::advance` after detonation as today. Capture all flame eliminations and revenge candidates before applying returns; create karts only for new non-final eliminations, resolve available returns, then run outcomes. Clear kart and temporary state if the round ends.

**Rationale**: `closing.rs::advance` emits `PatternStarted` only after movement/placement/detonation in the current loop. Checking only `closing.started` during input handling is one tick too late. Moving all closing processing earlier would change the documented rule that a bomb due on a block-landing tick still explodes. Returns must precede victory but cannot displace living survivors or delay a result while no safe square exists.

**Alternatives considered**: Wait until the first block lands (violates spec), perform returns next tick (wrong winner), or count karts/pending returns as survivors (prolongs rounds contrary to FR-014).

## 7. Deterministic revival and accounting

**Decision**: Select the lowest-slot qualifying victim as the preferred position; resolve owners in slot order. Scan safe squares by Manhattan distance, then row and column. Clear `eliminated_order` on return, preserve kills/bombs-placed match counters, reset ordinary powers and cooldown, and compute `bombs_active` from actual outstanding owned bombs rather than resetting it. Kart capacity checks only outstanding throws from the current stint.

**Rationale**: `state.rs` separates cumulative statistics, powers, and active counts. `bomb.rs` and `closing.rs` release active counts when bombs disappear. Erasing those counts on return would grant extra bombs. A return must not create temporary invulnerability or reuse a victim square while its killing flame is still active.

**Alternatives considered**: Random or original-corner respawns conflict with the chosen specification default; reset the whole participant (loses team/statistics/bomb ownership); stockpile returns (not permitted).

## 8. Additive snapshots without new event enum variants

**Decision**: Add serde-defaulted optional/neutral fields to participant, bomb, and flame observations; leave `ClientMsg::Input`, protocol envelopes, protocol version, and `Event` variants unchanged. Kart entry/motion/return, stun, and disappearance are visible in successive authoritative snapshots. Reuse compatible existing bomb/hop/landing/explosion/elimination events where semantically accurate; do not synthesize normal `Moved` events for a kart.

**Rationale**: `net/src/protocol.rs` wraps `Observation` and `Vec<Event>`. The actual `Event` enum in core has no unknown-variant fallback. Older serde decoders do not automatically ignore new enum variants, regardless of enum order; additive fields are the safe route. New clients decode old snapshots with no karts, no provenance, and no stun. Old clients can remain connected but cannot be promised a full kart presentation.

**Alternatives considered**: Add `KartSpawned` and similar variants (breaks old typed decoders), negotiate capabilities (unnecessary protocol expansion), derive eligibility client-side (violates authority). Backward wire decoding does not mean older renderers display the feature.

## 9. Inputs and bots

**Decision**: Keep the three-field `PlayerInput`. In physical-input adapters, neutralize simultaneous left/right before collapsing held directions when the participant is in a kart. Core accepts only `Left` or `Right` for kart movement, ignoring up/down/punch. JS, controller, touch, and local/online terminal paths retain existing bindings. Dispatch kart AI before the existing `!me.alive` no-op branch and preserve difficulty/reaction cadence.

**Rationale**: A single `mv: Option<Dir>` cannot encode two simultaneously held directions; core cannot reconstruct a conflict lost by an adapter. Browser and terminal currently forward actions without globally suppressing dead-player input, but the simulation and bot each reject dead participants. Tests must cover each actual reduction path rather than invent a new wire command.

`app.js::matchMenuModel` currently detects only bots remaining from human `alive` flags. Its special abandoned-match navigation must also consider a human kart as participating. The bot's `held` path also checks board passability at the death square; kart dispatch must bypass that path while retaining normal think cadence.

**Alternatives considered**: Add left/right booleans to network input (unneeded protocol work); make bots call private game state (cheating); change all normal movement conflict precedence (unrelated regression).

## 10. Reserve a real visual gutter

**Decision**: Fit one tile of outside-board gutter for the whole round in both renderers, including before any kart exists. Terminal size becomes at least `2 * (width + 2) + 2` columns and `(height + 2) + 2 + 4 + 1` rows, still bounded below by the existing minimums. WASM world bounds expand by one tile on each side in addition to the board front edge and pixel margin; the board's tiles and collision bounds do not change.

**Rationale**: `cli/src/view.rs::required_size` currently fits `width * 2 + 2` by `height + 2 + 4 + 1`. `web-render/src/view.rs` fits only `(0,0,width,height + EDGE)` with a 16-pixel margin. That fixed pixel margin is not enough for a one-tile vehicle at arbitrary scales. A status-only kart would fail the spatial gameplay requirement. Validate wrap/corner interpolation rather than interpolating a lap index through the entire board.

**Alternatives considered**: Draw only a status label (cannot aim spatially), overlay vehicles on playable tiles (misleading), rely on pixel margin (clipping), or resize when someone dies (layout jump). Small/medium/large minimum terminal sizes become 40x22, 44x24, and 56x28 respectively.

## Research Completion

No unresolved clarification remains. Package names and config values were verified directly. Explore reports were cross-checked against source where they disagreed: actual default arena is 15 x 13, `free_square` considers airborne bombs, the event enum has no unknown-variant escape hatch, and closing already participates after detonation. The design does not adopt recommendations that would limit returns to once per round or delay earned returns until the next tick.