# Data Model: Mobile Browser Gameplay

All state is transient presentation/input state in the browser page. No state is persisted or added to the match, player, server, or network protocol.

## Mobile Match Context

The conditions that determine whether mobile gameplay input is accepted.

| Field | Values | Rule |
|-------|--------|------|
| `stage` | Existing page stages | Touch gameplay is considered only while the existing stage is `playing`. |
| `touchCapable` | boolean | True when a coarse pointer, positive touch-point count or touch-event support is available. |
| `orientation` | `landscape`, `portrait` | Derived from the current viewport orientation. |
| `pageActive` | boolean | False while hidden or the browser page has lost focus. |
| `navigationOpen` | boolean | True while an in-match navigation/settings layer is open. |
| `layout` | portrait or landscape | Portrait uses bottom-corner controls; landscape retains the existing horizontal presentation. |
| `gameplayEnabled` | boolean | True while playing with no navigation layer and an active page; orientation does not block input. |

When `gameplayEnabled` becomes false, all touch, keyboard and controller gameplay input is cleared and the next input message is neutral. An orientation change clears held input before applying the new layout.

## Joystick Gesture

The active gesture in the left-side movement zone.

| Field | Type | Rule |
|-------|------|------|
| `pointerId` | pointer identifier or null | Identifies the one pointer currently owning joystick movement. |
| `origin` | x/y position or null | Position where the gesture began; used as the neutral point. |
| `current` | x/y position or null | Latest position for the owning pointer. |
| `direction` | `Up`, `Down`, `Left`, `Right`, or null | The selected cardinal direction; null inside the central neutral area or when inactive. |

Transitions: inactive -> tracking on a valid pointer down; tracking -> updated direction on pointer movement; tracking -> neutral on pointer up, cancellation, lost capture, focus loss or orientation change. Only the owning pointer can update or release this gesture.

## Action Press

A one-shot action initiated in the right-side zone.

| Field | Type | Rule |
|-------|------|------|
| `action` | `bomb` or `punch` | Corresponds to the existing player action. |
| `pointerId` | pointer identifier | Identifies the independent touch assigned to the action control. |
| `pressed` | boolean | True only for the input tick containing a new valid press; holding a finger does not repeat the action. |
| `visible` | boolean | Bomb is visible during mobile play; Punch is visible only while the current player's observation has `has_punch`. |

A punch press uses existing game behavior and has an effect only when the player has the punch power. Cancelled or gameplay-disabled presses do not enqueue a later action.

## Effective Player Input

The normalized values combined for one existing client input tick.

| Field | Type | Rule |
|-------|------|------|
| `mv` | optional cardinal direction | Selected controller direction takes precedence; otherwise active joystick direction takes precedence over the existing keyboard direction. |
| `bomb` | boolean | One-shot press from touch, selected controller, or keyboard for this tick. |
| `punch` | boolean | One-shot press from touch, selected controller, or keyboard for this tick. |

The existing client message, player slot assignment, server validation and game rules are unchanged.

## Round Result

Transient client state retained after the server ends a round.

| Field | Type | Rule |
|-------|------|------|
| `outcome` | winner, team/chaser result, or draw | The exact outcome from the server's `Ended` message. |
| `finalObservation` | last game observation | Captures final alive state, team/role and available player stats from the final snapshot. |
| `lobby` | latest lobby view | May update after the round ends but does not dismiss the result view. |
| `visible` | boolean | True after `Ended`; remains true until Return to Lobby is activated. |

## Final Scoreboard Row

Derived from the result's final observation and the final lobby slot names; no new score or kill-count rules are introduced.

| Field | Type | Rule |
|-------|------|------|
| participant | name/slot | Every occupied slot is included. |
| standing | Winner, Survived, Eliminated, Draw, or mode-specific result | Derived from outcome, alive state, team and role. |
| stats | bombs, range, speed and active powers | Taken from the final participant observation; no values are invented when data is missing. |
