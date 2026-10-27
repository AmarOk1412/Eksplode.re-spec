# Data Model: Mouse-Free Game Navigation

Runtime presentation state only. Nothing is persisted, sent to the server or added to the game state.

## Navigation Context

What the current input means.

| Field | Type | Rule |
|-------|------|------|
| stage | `idle`, `joining`, `lobby`, `playing`, `closed` | Existing client stage; `playing` with no open layer means gameplay input |
| layers | ordered list of Layer | Top layer receives navigation; an empty list means the stage view |
| mode | `navigation` or `gameplay` | `gameplay` only when stage is `playing` and no layer is open |

Transitions: opening or closing a layer, or a stage change, resets controller edge state (every button must be released once) and clears held gameplay input.

## Layer

An overlay that traps focus.

| Field | Type | Rule |
|-------|------|------|
| id | `hub`, `sound`, `controls`, `osk` | Unique; at most one of each open |
| root | element | Everything outside it is inert while it is the top layer |
| opener | element or null | Focus returns here on close if it is still enabled, else to the first enabled control of the layer below |
| initialFocus | element | First enabled control of the layer |

## Focus Target

A control that can receive focus.

| Field | Type | Rule |
|-------|------|------|
| element | element | Visible, not disabled, not inert, with a non-empty bounding rectangle |
| kind | `button`, `checkbox`, `select`, `range`, `radio`, `text` | Decides what left, right and A do |
| key | string | Stable `id` or `data-` value used to restore focus after a re-render |

## Navigation Input

One normalized input from the keyboard or controller.

| Field | Values | Source |
|-------|--------|--------|
| action | `up`, `down`, `left`, `right`, `confirm`, `back`, `menu` | Controller: D-pad or left stick, A, B, Start. Keyboard: arrows on buttons, Enter or Space, Escape |
| repeat | boolean | Directions repeat at 400 ms then every 120 ms while held; the other actions fire once |

Validation: the stick presses at 0.5 and releases at 0.3; missing or short Gamepad arrays count as neutral.

## On-Screen Keyboard State

| Field | Type | Rule |
|-------|------|------|
| target | text field | The field being edited |
| original | string | Restored on cancel |
| buffer | string | Never longer than the field's `maxlength` (player name 16, room name 24); control characters are dropped |
| shift | boolean | Upper or lower case letters |
| cursor | row and column | Always on an enabled key |

Actions: character, space, backspace, shift, confirm (write the buffer back and fire `input`), cancel (restore the original). An empty player name is allowed in the buffer; the existing join validation still rejects it.
