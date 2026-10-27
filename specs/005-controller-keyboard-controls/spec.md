# Feature Specification: Controller and Keyboard Controls

**Feature Branch**: `005-controller-keyboard-controls`

**Created**: 2026-10-27

**Status**: Draft

**Input**: User description: "Players should be able to play with an xbox controller and configure the keyboard touches like they want"

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Play with an Xbox-compatible controller (Priority: P1)

A player using the web game can use an Xbox-compatible controller to move, place bombs and use the punch action, without needing a keyboard during a match. The controller works alongside existing keyboard play, and a player can return to keyboard input at any time.

**Why this priority**: Controller support is the primary access need in the request and lets players use the input device they prefer.

**Independent Test**: Connect a supported controller, join a match, and verify movement, bomb placement and punch actions; disconnect it and verify keyboard input continues to work.

**Acceptance Scenarios**:

1. **Given** a supported controller is connected before or during a match, **When** the player uses its directional controls, **Then** their character moves in the corresponding direction.
2. **Given** a supported controller is connected, **When** the player presses the assigned bomb or punch control, **Then** the corresponding game action is sent once according to the game's existing input rules.
3. **Given** the controller disconnects or becomes unavailable, **When** the player uses the keyboard, **Then** keyboard play continues without restarting or leaving the match.
4. **Given** multiple controllers are connected, **When** a player selects a controller, **Then** only that controller supplies their game actions.

### User Story 2 - Customize keyboard controls (Priority: P1)

A player can change the keyboard keys for movement, bomb placement and punch to suit their preference. The player can see the current assignments, replace an assignment, restore the defaults and keep their choices for later visits.

**Why this priority**: Custom keyboard controls are the other explicit request and help players whose preferred layout differs from the defaults.

**Independent Test**: Change each action's key, play a match using the new keys, reload or reopen the game, verify the assignments remain, then restore and verify the defaults.

**Acceptance Scenarios**:

1. **Given** a player opens control settings, **When** they choose an action and press a new available key, **Then** the displayed assignment changes and that key performs the action in a match.
2. **Given** a web player has changed assignments, **When** they return to control settings in a later visit on the same device, **Then** their assignments are still shown and usable.
3. **Given** a player restores default controls, **When** they confirm the reset, **Then** the standard assignments are active again.
4. **Given** a player is using the terminal version, **When** they customize keyboard assignments for their player slot, **Then** the changed assignments work for that slot without changing the other slots' assignments.
5. **Given** a player joins a network game from the terminal, **When** they provide custom keyboard assignments at launch, **Then** only their own controls use those assignments.

### Edge Cases

- A player assigns the same key to conflicting actions: the game rejects the conflict with a clear explanation and leaves the previous assignments unchanged.
- A player presses Escape, a browser-reserved key or a key already used by the interface while assigning a control: the game does not silently capture it as a gameplay binding and explains the restriction.
- A controller is connected after the match begins or is disconnected while an action is held: the game detects the change and does not leave movement or actions stuck.
- A controller is not recognized or is not supported by the browser: keyboard controls remain available and the player receives a clear status message.
- A saved keyboard assignment is invalid, unavailable or from an older version: the game safely uses defaults for the affected assignment and allows the player to change it.
- Two local terminal player slots are assigned the same key: the game prevents ambiguous assignments or clearly identifies which slot will receive the action.
- The player changes bindings while a match is active: the new bindings take effect without interrupting the match, but do not alter actions already in progress.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: The web game MUST accept input from Xbox-compatible game controllers for movement, bomb placement and the punch action.
- **FR-002**: The controller MUST have clearly visible default assignments for each supported action; the directional pad and analog stick MUST both support movement.
- **FR-003**: The game MUST allow a player to select which connected controller controls their player when multiple controllers are available.
- **FR-004**: The game MUST detect controller connection and disconnection during play and MUST continue to accept keyboard input when no controller is available.
- **FR-005**: The game MUST provide a control settings view where a player can inspect and change keyboard assignments for movement, bomb placement and punch.
- **FR-006**: The game MUST retain the existing default keyboard assignments and provide a reset-to-default action.
- **FR-007**: Keyboard assignments configured in the web control panel MUST be saved per player on that player's device and restored on a later visit; invalid saved assignments MUST fall back safely to defaults.
- **FR-008**: The terminal version MUST allow keyboard assignments to be configured independently for each local player slot at launch and allow a network-joining player to configure their own assignments; it MUST document how to configure them.
- **FR-009**: The game MUST reject or resolve conflicting assignments without silently changing another action's meaning, and MUST explain the conflict to the player.
- **FR-010**: Control settings MUST be available outside a match; changing bindings during a match MUST not interrupt the match or leave an action held.
- **FR-011**: Keyboard, controller and terminal input MUST produce the same existing movement, bomb and punch actions, without changing game rules or multiplayer authority.
- **FR-012**: The documentation MUST describe controller support, default controller assignments, keyboard customization, persistence and restoring defaults.

### Key Entities *(include if data involved)*

- **Control assignment**: The relationship between an input (keyboard key or controller control) and a game action for a player.
- **Player control preference**: A player's saved keyboard assignments and selected controller preference on their device.
- **Connected controller**: A game controller currently available to the web game, identified separately when more than one is present.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: A player can complete a match's movement, bomb and punch actions using an Xbox-compatible controller without using the keyboard.
- **SC-002**: A player can change all keyboard action assignments and complete a match using those assignments in under two minutes.
- **SC-003**: In 100% of tested reloads, valid saved keyboard assignments are restored; invalid assignments fall back to usable defaults without preventing play.
- **SC-004**: In 100% of tested controller disconnections, keyboard input remains usable and no action remains stuck.
- **SC-005**: In 100% of tested conflicting key assignments, the conflict is reported and the existing valid mapping is preserved.
- **SC-006**: Players can find instructions for controller and keyboard controls in the project documentation within two minutes.

## Assumptions

- Xbox-compatible controllers are supported in the graphical web game; the terminal game remains keyboard-driven.
- Standard controller assignments cover the game's existing movement, bomb and punch actions. Controller assignment customization is not required in this feature; keyboard assignments are customizable.
- Web keyboard preferences are local to the player's device and do not change another player's controls or require sharing through the multiplayer room; terminal key assignments apply to the current CLI run, and a joining client can only configure their own controls.
- Existing keyboard defaults remain unchanged until a player chooses a different mapping.
- The current game has four local keyboard player slots in terminal play; each slot's configuration is independent.
- Reserved browser and operating-system shortcuts may not be assignable where the game cannot reliably receive them.