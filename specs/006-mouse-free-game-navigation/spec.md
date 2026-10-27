# Feature Specification: Mouse-Free Game Navigation

**Feature Branch**: `006-mouse-free-game-navigation`

**Created**: 2026-10-27

**Status**: Draft

**Input**: User description: "All pages of the games, including lobby, settings MUST be controllable without a mouse (via game controller or keyboard)."

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Navigate the web game with a keyboard (Priority: P1)

A player can open the web game and complete the normal flow without using a mouse: create or join a room, enter their name, choose a character, operate the lobby, adjust available settings, and play or return to the lobby. Every visible, enabled control has a clear keyboard focus position, and the player can always tell which control will receive the next action.

**Why this priority**: Keyboard access must work even when no controller is available, so it is the baseline mouse-free path for every web player.

**Independent Test**: From a fresh page load, use only Tab/Shift+Tab, arrow keys, Enter/Space and Escape to create or join a room, enter text, change a setting, reach the lobby, start a match, open and close settings, and return to the lobby.

**Acceptance Scenarios**:

1. **Given** the page has loaded, **When** a player presses Tab, **Then** visible keyboard focus moves to the first available control and continues through enabled controls in a predictable order; Shift+Tab moves backwards.
2. **Given** a button, link-like action, checkbox, slider, select list or choice group has focus, **When** the player presses its standard activation or adjustment key, **Then** it performs the same action as a pointer activation without requiring a mouse.
3. **Given** the player moves between room setup, join, character editing, lobby, sound settings, control settings and the match, **When** each view appears, **Then** the current view has a visible focus position and hidden or disabled controls are skipped.
4. **Given** a text field is focused, **When** the player types or edits text with the keyboard, **Then** the field accepts the same valid input as it does with pointer use and the game does not interpret those keystrokes as gameplay actions.
5. **Given** a panel or dialog is open, **When** the player presses Escape, **Then** it closes or returns to its parent view and focus returns to the control that opened it; Escape never exits the browser or unexpectedly leaves a match.
6. **Given** a match is active, **When** the player uses existing movement, bomb and punch keys, **Then** gameplay controls continue to work; when a settings panel or text field has focus, navigation and editing keys do not trigger gameplay actions.

### User Story 2 - Navigate the web game with an Xbox-compatible controller (Priority: P1)

A player can complete the same room, character, lobby and settings flow using a connected standard-mapped Xbox-compatible controller without touching a mouse or keyboard. Menus show which item is focused, and text fields can be completed using an on-screen keyboard.

**Why this priority**: Controller support must extend beyond the match itself; otherwise a player who starts with a controller is forced to use another device to reach the game.

**Independent Test**: From a fresh page load, use only the selected controller to create or join a room, enter a player name, choose a character, change a room setting, start a match, open and close settings, and return to the lobby.

**Acceptance Scenarios**:

1. **Given** a supported controller is selected and a web view is active, **When** the player uses the D-pad or left stick, **Then** visible focus moves between enabled controls and choice groups in a predictable order.
2. **Given** a control has focus, **When** the player presses Xbox A, **Then** the focused control is activated; when the player presses Xbox B, **Then** the current panel is cancelled or closed and focus returns to its parent.
3. **Given** a player focuses a text field using the controller, **When** they activate it, **Then** an on-screen keyboard allows them to enter, edit, confirm or cancel text using only the controller.
4. **Given** a player is in a settings panel or on-screen keyboard, **When** they navigate or activate controls, **Then** those actions do not place a bomb, punch a bomb or otherwise trigger gameplay input.
5. **Given** a controller is used during active play and no navigation panel is open, **When** the player uses its gameplay controls, **Then** existing movement, bomb and punch mappings remain active; opening the navigation/settings interface takes focus from gameplay and leaving it clears held inputs.
6. **Given** the selected controller disconnects while a menu is open, **When** the player continues with the keyboard, **Then** the page remains operable, focus stays on a valid control, and no action remains held.

### Edge Cases

- The page initially receives focus on a non-interactive area: the next Tab or controller navigation action moves focus to the first enabled control without requiring a pointer click.
- A view contains no enabled controls or every available control is disabled: navigation remains stable and does not target hidden or disabled elements.
- A controller is connected after the page opens, loses focus, disconnects, reconnects or is replaced: menu focus and held actions recover without selecting or activating a stale control.
- A player moves between a text field and gameplay: text-entry keys are not interpreted as game actions, and gameplay input is cleared when leaving the field or panel.
- A controller's confirm/back buttons are held while a view changes: the new view does not immediately activate or dismiss another control.
- A settings panel, confirmation prompt or on-screen keyboard is open over a match: gameplay actions remain suppressed until the player returns to the match.
- A browser does not expose a compatible controller: all web screens remain fully keyboard-operable and the player receives a clear controller status.
- Mouse input is used partway through a keyboard/controller flow: it remains supported and the visible focus follows the activated control.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: Every visible, enabled interactive control in the web game MUST be reachable and operable without a mouse, including room creation, joining, name and character entry, lobby actions, room settings, sound settings, control settings, in-match panels and optional developer controls.
- **FR-002**: The web game MUST provide keyboard navigation in a predictable visual order, support forward and backward focus movement, and show a clear visible focus indicator on the active control.
- **FR-003**: The keyboard MUST be able to activate buttons, toggle checkboxes, adjust sliders, operate select lists and move within choice groups using standard keys; Escape MUST close or return from the active panel without unexpectedly leaving the match.
- **FR-004**: Keyboard focus MUST move to an appropriate control when a view or panel opens, skip hidden and disabled controls, and return to the originating control when a panel closes.
- **FR-005**: The selected standard-mapped controller MUST be able to navigate every web view using the D-pad or left stick, activate the focused control with A, and cancel or go back with B.
- **FR-006**: The controller MUST have a consistent way to open and close the web navigation/settings interface from the join flow, lobby, settings and active match without mouse input.
- **FR-007**: A player using only a controller MUST be able to enter and edit text in every required text field, including player and room names, through an on-screen keyboard with confirm and cancel actions.
- **FR-008**: Navigation, settings and text-entry input MUST NOT trigger gameplay movement, bomb or punch actions; leaving a panel or text-entry control MUST clear any held gameplay input.
- **FR-009**: When no controller is available or the selected controller disconnects, keyboard navigation MUST remain usable and no focus or input state may become stuck.
- **FR-010**: Controller navigation in views MUST remain separate from in-match gameplay mappings: menu confirmation/back actions apply while navigating, and existing gameplay mappings apply while playing outside navigation panels.
- **FR-011**: Keyboard and controller navigation MUST work with optional panels and settings enabled or hidden according to the current room, player role and developer configuration.
- **FR-012**: The documentation MUST describe keyboard navigation, controller menu navigation, controller text entry and mouse-free access to the game flow.

### Key Entities *(include if data involved)*

- **Focused control**: The visible, enabled action or field that will receive the next keyboard or controller activation.
- **Navigation context**: The active web view or overlay (such as join, lobby, settings, confirmation, text entry or match) that determines focus order and whether inputs mean navigation or gameplay.
- **On-screen keyboard**: A selectable set of characters and editing actions used to enter text when the player has a controller but no keyboard.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: In 100% of tested web flows, a player can move from a fresh page load through room entry, lobby setup, match start and return to the lobby using only a keyboard.
- **SC-002**: In 100% of tested web flows, a player can complete the same path using only a supported controller, including player-name and room-name entry.
- **SC-003**: Every visible, enabled control in the web interface is reachable and operable through keyboard focus and controller navigation in the acceptance test matrix.
- **SC-004**: In 100% of tested view and panel transitions, the focus indicator identifies an enabled control and hidden or disabled controls receive no action.
- **SC-005**: In 100% of tested menu and text-entry actions, no bomb or punch action is triggered; after returning to play, gameplay input resumes without a stuck or repeated action.
- **SC-006**: Players can discover how to navigate all web screens without a mouse from the project documentation within two minutes.

## Assumptions

- This feature covers the graphical web game. The terminal version already operates without a mouse and retains its current controls.
- Keyboard navigation uses standard browser keyboard behavior where appropriate, with visible game-specific focus and explicit handling for menus, choice groups and panels.
- The Xbox-compatible controller follows the standard controller mapping supported by the existing gamepad feature; controller navigation uses D-pad/left stick, A to activate and B to cancel or go back.
- A menu/navigation button on the selected controller provides a consistent way to open the navigation/settings interface from the active match; gameplay mappings continue when no navigation view is active.
- The on-screen keyboard is only needed when text is entered through a controller; players using a physical keyboard continue to type directly into text fields.
- Mouse support remains available; mouse use is optional and is not removed or restricted.
- This feature changes presentation navigation only. It does not change match rules, server authority or network messages.