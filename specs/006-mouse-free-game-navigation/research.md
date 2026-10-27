# Research: Mouse-Free Game Navigation

## Focus Model

- **Decision**: Use real DOM focus (`document.activeElement`) as the single source of truth for both keyboard and controller. Controller input only moves focus and activates the focused element.
- **Rationale**: Native focus gives Tab order, screen-reader support and the existing `:focus-visible` styling for free, and mouse clicks keep the focus in step. A virtual cursor would need its own highlight and would diverge from keyboard focus.
- **Alternatives considered**: A virtual cursor drawn by the page (duplicates focus state); `tabindex` roving everywhere (large markup churn).

## Direction Movement

- **Decision**: Move focus to the nearest enabled control in the pressed direction using element rectangles, falling back to DOM order when none lies in that direction. Tab and Shift+Tab always follow DOM order. For the keyboard, arrow keys use the same movement only on controls with no native arrow behavior (buttons); selects, sliders and radio groups keep their native arrows.
- **Rationale**: Panels mix rows of buttons, selects and card grids, so spatial movement is more natural than a flat list, and it is a pure function of rectangles that Node tests can cover with fake data.
- **Alternatives considered**: DOM order only (up and down would jump unpredictably in the lobby button row); explicit neighbour tables per screen (fragile as markup changes).

## Adjustable Controls With A Controller

- **Decision**: With the controller, left and right on a focused select cycle its value and dispatch `change`; on a range they step the value and dispatch `input` and `change`; on a radio group they move the selection; A toggles a checkbox and presses buttons; A on a text field opens the on-screen keyboard. Up and down always leave the control.
- **Rationale**: A native select popup cannot be driven by a gamepad, and cycling values is the usual console menu behavior. Dispatching the same events keeps the existing handlers (`changeRoom`, `editRoom`, audio settings) unchanged.
- **Alternatives considered**: Custom replacement widgets for every select (large rewrite, new tests); popup lists (extra layer for four-value choices).

## Layers And Focus Trap

- **Decision**: A small layer stack (hub, panel, on-screen keyboard). While a layer above the view is open, everything outside it is made inert; opening records the opener, and closing restores focus to it. Escape or B closes the top layer; with no layer open, Escape or Start opens the menu hub.
- **Rationale**: The sound and controls panels are fixed overlays, not modal; without a trap, directional movement would escape into the page behind. `inert` removes covered controls from focus and clicks with no custom code.
- **Alternatives considered**: Native `<dialog>` with `showModal()` (changes markup, CSS and existing tests for little gain); manual Tab handlers only (misses controller movement).

## Controller Navigation Input

- **Decision**: A `NavTracker` separate from the gameplay `GamepadTracker`. It reads the same selected device, maps D-pad and left stick (press at 0.5, release at 0.3) to four directions with repeat (400 ms first, then every 120 ms), and maps A, B and Start to confirm, back and menu with single fire. When the context changes (view change, layer opened or closed) every button must be released once before input is accepted again.
- **Rationale**: Gameplay uses a lower dead zone and one-shot actions; menus need edge detection with repeat. The release-first rule prevents the A press that opens a panel from also activating the first item, which is the edge case in the spec.
- **Alternatives considered**: Extending `GamepadTracker` (mixes two meanings of the same buttons); reusing the 0.25 dead zone (accidental menu drift).

## Routing Between Menu And Gameplay

- **Decision**: Input means navigation whenever the stage is not `playing` or any layer is open; otherwise it means gameplay. Entering navigation clears held gameplay input; leaving it clears navigation state so no action is repeated. The 50 ms tick now samples the controller in every stage and routes the result.
- **Rationale**: The current tick returns before sampling unless a match runs, so the controller cannot drive menus. A single routing rule satisfies FR-008 and FR-010.
- **Alternatives considered**: A second polling interval (extra loop, risk of both consuming the same press).

## On-Screen Keyboard

- **Decision**: A layer made of real buttons in a grid (letters, digits, space, a few safe punctuation marks, shift, backspace, confirm, cancel), driven by a pure reducer that applies the field's `maxlength` and drops control characters. Confirm writes the text back and fires the field's `input` event; cancel restores the original text.
- **Rationale**: Buttons reuse the navigation and focus logic; a pure reducer is testable; limits match the host's validation for names (1 to 16 characters for players, 24 for rooms), so the server stays the authority.
- **Alternatives considered**: Letter-wheel entry (slow, new interaction model); browser or OS on-screen keyboards (not available on desktop Linux).

## Re-Render Safety

- **Decision**: Preserve focus across re-renders by remembering the focused element's stable key (`id` or `data-` attribute) before `updateUi` and re-focusing it after. The room creation menu rebuilds its mode buttons on every edit and must keep the focused one.
- **Rationale**: `replaceChildren` drops focus, which would send keyboard and controller users back to the start of the panel after each change.
- **Alternatives considered**: Rebuilding only changed attributes (larger refactor of the existing menus).

## Menu Hub And Back Behavior

- **Decision**: The hub lists Sound, Controls and Resume and is reachable from every screen by Start or Escape. Escape or B closes the top layer and never leaves a room or match. In controls key capture, Escape cancels the capture instead of being reported as a reserved key.
- **Rationale**: FR-006 requires one consistent way to reach settings from the join flow, the lobby and a running match, and keyboard users need the same entry. The match keeps running (the host is authoritative); gameplay input is suppressed while a layer is open, as already done for the controls panel.
- **Alternatives considered**: Pausing the match (not possible in a shared match); the Back button as the opener (Start is the common menu button).

## Test And Hardware Strategy

- **Decision**: Node tests cover rectangle navigation, the layer stack, controller edges and repeat, adjustable controls and the on-screen keyboard reducer. The smoke script walks the whole flow with CDP keyboard events and the synthetic Gamepad harness from feature 005. A physical controller check follows the same Linux procedure as before.
- **Rationale**: Most behavior is pure; real devices only prove browser exposure, which feature 005 found to be environment dependent (its physical check, task T027, is still open).
- **Alternatives considered**: Hardware-only verification (cannot run in CI).
