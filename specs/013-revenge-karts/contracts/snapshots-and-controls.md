# Snapshot and Control Contract: Revenge Karts

**Status**: Proposed additive contract | **Related**: [data model](../data-model.md), [gameplay](gameplay.md)

## Wire Envelope

Keep protocol version 1 and existing TCP/WebSocket room envelopes, `ClientMsg::Input`, `ServerMsg::Snapshot`, and event variants. Versioned multi-room routing remains unchanged. Input remains:

```json
{"type":"Input","tick":241,"mv":"Right","bomb":true,"punch":false}
```

The host resolves the sender's slot as today. Clients cannot submit kart position, stun duration, source provenance, pending returns, revived state, or another participant's actions. Existing stale/future tick and malformed-input validation applies unchanged.

## Additive State Fields

Field names and defaults:

| Object | Field | Neutral Value | Non-neutral Shape |
|--------|-------|---------------|-------------------|
| Participant | `kart` | absent or `null` | Object with `index`, `entered_tick`, `cooldown_left`, `travel`, `pending_return`. |
| Participant | `immobilized_left` | absent or `0` | Unsigned movement-pass countdown, at most `tick_hz`. |
| Bomb | `revenge_stint` | absent or `null` | Unsigned round tick identifying the throwing kart stint. |
| Flame | `revenge_stints` | absent or `[null,null,null,null]` | Exactly four nullable entry ticks, indexed by bomb owner slot. |

Example additions to an otherwise unchanged participant object:

```json
{
  "alive": false,
  "kart": {
    "index": 7,
    "entered_tick": 240,
    "cooldown_left": 0,
    "travel": "Right",
    "pending_return": null
  },
  "immobilized_left": 0
}
```

This is a fragment, not a full snapshot fixture; all existing required participant fields remain. `travel` accepts only `Left`, `Right`, or null for a kart. Top/Right/Bottom/Left edge and inward aim are derived from dimensions/index rather than independently serialized. A pending return is a `Pos` object `{"x":7,"y":4}`, not a granted spawn or a reserved position.

## Compatibility Rules

1. Deserialize absent new fields with neutral defaults. New renderer/client code must accept existing committed observation fixtures and legacy host snapshots.
2. Older peers may ignore added object fields; do not emit any new `Event` enum variant. The current core enum has no unknown-variant fallback, so adding a variant would break old typed snapshot decoding.
3. Full kart presentation is guaranteed only for updated clients. Old clients can remain connected with their old dead-player rendering; wire compatibility is not a claim that unmodified clients can display karts or stun.
4. Emit existing `BombPlaced`, `BombHopped`, `BombLanded`, `BombExploded`, and `PlayerEliminated` events when those actual events occur. Do not forge board `Moved` events or an additional `PlayerEliminated` on cutoff. New presentation transitions derive from consecutive authoritative snapshots and can be missed without corrupting current state.
5. On first snapshot, reconnect, or a gap, render current state directly rather than replaying a guessed return/entry sequence. Reset interpolation for living-to-kart and kart-to-living transitions.
6. Check invalid indices, impossible alive-plus-kart combinations, future source ticks, incorrect provenance-array lengths, and unexpected travel values at deserialization/view validation boundaries. Reject malformed typed payloads or suppress invalid added entities safely; never panic or index arrays blindly. Clients' observations are not accepted as authoritative host input.
7. Preserve the existing 64 KiB message limit. Test serialized snapshots in worst-case supported-room situations with four kart owners and overlapping source-bearing flames, without raising the limit.

## Physical Input Reduction

| Participation | Movement | Bomb | Punch |
|---------------|----------|------|-------|
| Living | Existing directional behavior, blocked while immobilized | Existing placement rules | Existing punch rules |
| Kart | Existing left/right bindings mean counterclockwise/clockwise; up/down ignored | Inward throw | Ignored |
| Eliminated without kart or ended round | No action | No action | No action |

`PlayerInput.mv` contains at most one direction. For a kart, adapters must inspect left/right held state before last-key or D-pad precedence and produce null when both are held. Also normalize a left request from one active input source and right from another to neutral before combining sources; retain existing source precedence for ordinary alive controls. Preserve bomb button edge-trigger/repeat behavior already provided by each device and clear held actions on focus/menu/controller changes as today.

Keyboard remaps use the existing left/right and bomb actions. A standard gamepad uses the horizontal stick or left/right D-pad; a simultaneous opposing D-pad pair is neutral, and up/down cannot override a valid horizontal kart intent. Touch uses the existing joystick's horizontal intent and bomb action. For terminal input, retain per-direction press/repeat/release or bounded hold information long enough to resolve opposing directions per tick; do not assume terminals universally provide key-release events. No new key, binding preference, or network field is required.

Core still ignores illegal kart directions and validates capacity/phase/eligibility; it cannot detect physical conflicts already collapsed into a single valid wire direction.

## Browser State and Navigation

- Include kart and stun fields in participant row/view models instead of showing an active kart as only dead. Keep identity, colour, team, and visible return transition; show stun via a restrained visible effect and remaining state, not a modal.
- The current browser menu's `onlyBots`/no-human-survivors test must count a human with a kart as still participating. Do not offer the special abandoned-match return path solely because all human `alive` flags are false while a human kart remains. Existing explicit leave/menu behavior remains available.
- Gameplay focus and touch controls remain available during kart participation. An ended round or participant without a kart cannot send effective gameplay actions.
- The client never computes its own return square, hit outcome, or cutoff. It displays the latest authoritative state.

## Spatial Rendering

Reserve a one-tile visual gutter on all four sides for the entire round in both updated frontends; it is not part of `Arena` and does not change tile coordinates. The virtual ring maps onto an outside rectangle, with centers at half-tile offsets beyond the board and sprites fitting within the gutter. Render same-position karts with deterministic small visual offsets or layered slot markers so all owners remain identifiable; these offsets must never alter authoritative position or targeting.

**Terminal**: Draw the actual kart perimeter positions using two-character vehicle/slot glyphs and player colours, with team indicators retained in status lines. Do not replace spatial drawing with a status-only kart. Update `required_size`, board/title placement, and status offsets consistently. Minimum supported sizes after the fixed gutter are small 40x22, medium 44x24, large 56x28. Exact-size terminals must render without truncation; smaller terminals use the existing required-size notice. Mark stunned survivors distinctly without changing cell width.

**WASM/canvas**: Expand fit bounds to include the gutter, board front edge, and existing pixel margin. Draw a compact kart sprite with the participant's visible identity/team, aimed inward. Interpolate adjacent perimeter indices along the perimeter, including the final-to-zero wrap and corner bends; never through the board or the long way around a lap. Snap directly on entry, return, reconnect, or incompatible state changes. Keep the viewport stable when karts appear/disappear. Check desktop and mobile canvas bounds and readable status text.

No new audio IDs or network events are required. Existing bomb and round sounds remain; optional cosmetic changes must not add gameplay rules to JavaScript.

## Required Contract Tests

- Old fixtures decode to `kart = None`, zero stun, and no revenge provenance.
- New snapshot round trips preserve every new field and remain decodable after removing only added fields with the legacy schema/event list.
- TCP and WebSocket observers receive identical kart/stun/provenance/outcome state for the same tick.
- Boundaries reject or safely ignore invalid new state, without panics; forged client input cannot assign a kart or return.
- Keyboard remapping, D-pad conflicts, cross-device conflicts, touch diagonals, terminal held-key expiry, neutral inputs, and focus loss match the control table.
- Browser no-human-alive navigation remains in-match for human karts and follows existing behavior after every human is permanently eliminated.
- Terminal and canvas projections agree with the shared edge/lane helper at each corner; fitting and snapshots include all four karts, even at the same track index.