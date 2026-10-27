# Navigation Controls Contract

## Keyboard

- Tab and Shift+Tab move through enabled, visible controls in page order; focus is visibly marked on every control type (buttons, text fields, selects, sliders, checkboxes, mode and colour choices, on-screen keys).
- Enter or Space activates the focused button and toggles a checkbox; native keys adjust selects and sliders.
- Arrow keys move focus between buttons spatially; on selects, sliders and radio groups they keep their native behavior, and a mode or colour choice group moves its selection with left and right.
- Escape closes the top layer and returns focus to its opener. With no layer open, Escape opens the menu hub. Escape never leaves a room or match.
- Inside controls key capture, Escape cancels the capture.
- Typing in a text field is text entry only; gameplay keys are ignored while a field or layer has focus.

## Controller (standard mapping, selected device)

| Input | Meaning in navigation |
|-------|----------------------|
| D-pad or left stick | Move focus up, down, left or right (stick presses at 0.5, releases at 0.3; repeat after 400 ms, then every 120 ms) |
| A | Activate the focused control; toggle a checkbox; open the on-screen keyboard on a text field |
| B | Back: close the top layer, or cancel text entry |
| Start | Open the menu hub; close it when it is the top layer |
| Left or right on a select | Cycle to the previous or next option and fire `change` |
| Left or right on a slider | Step the value and fire `input` and `change` |
| Left or right on a radio group | Move the selection |

- A button held when the context changes is ignored until released.
- Navigation input never produces bomb, punch or movement messages. In a match with no layer open, the gameplay mapping from feature 005 applies unchanged.
- If the controller disconnects, keyboard navigation continues and focus stays on an enabled control.

## Menu Hub

- Lists Sound, Controls and Resume; reachable from the room menu, join screen, lobby and match by Start or Escape.
- Sound and Controls open their panels as layers above the hub; closing a panel returns to the hub; Resume or back closes the hub.
- The match keeps running while any layer is open; gameplay input is suppressed until all layers are closed.

## On-Screen Keyboard

- Opens from a focused text field with A. Rows of letters, digits, space and a few punctuation marks, plus shift, backspace, confirm and cancel keys, all reachable by normal navigation.
- Respects the field's `maxlength` and drops control characters. Confirm writes the text back and fires `input`; cancel or B restores the original text. Focus returns to the field.

## Screens In Delivery Order

1. **Settings**: menu hub, sound panel (two sliders, mute, test, close), controls panel (six key bindings, controller selector, restore defaults with confirmation, close).
2. **Lobby**: room name (on-screen keyboard), mode, board, time, friendly fire, slot actions (start, add and remove bot, ready, change look), character editor (colour swatches, look, head and face selects, random, done).
3. **Main screen**: room creation (name, mode choices, board, time, friendly fire, create), join (player name, Join, Retry), match (menu hub access, game panel), developer panel (pause, reset, drop sliders) when enabled.

## Boundaries

- No server message, game rule, core or terminal change.
- Mouse use stays supported and focus follows the control that was clicked.
