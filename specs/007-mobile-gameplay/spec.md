# Feature Specification: Mobile Browser Gameplay

**Feature Branch**: `007-mobile-gameplay`

**Created**: 2026-10-27

**Status**: Draft

**Input**: User description: "On mobile, stay vertical: joystick at bottom left and bomb/punch at bottom right. If horizontal, keep the current layout. Punch is shown only while the player has Punch. Firefox for Android must work. Show the scoreboard before the player returns to the lobby."

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Join and manage a game on a phone (Priority: P1)

A player opens the game address in a phone browser and uses the touchscreen to enter a name, create or join a room, choose their character, use the lobby and room settings, and start or join a match. The interface adapts to the available screen space and does not depend on hover or a physical keyboard.

**Why this priority**: A phone player must be able to reach a match and manage the ordinary game flow without switching to a computer.

**Independent Test**: Using only a phone touchscreen, open the game, enter a name, join or create a room, change an available setting, and reach a started match.

**Acceptance Scenarios**:

1. **Given** a player opens the game address in a supported phone browser, **When** the page loads, **Then** the join or room-creation view fits the screen and its controls can be activated by touch.
2. **Given** the player is in the join flow or lobby, **When** they enter a name, choose a character, adjust an available room setting, or start a match, **Then** each action can be completed using the phone's touchscreen and text-entry keyboard.
3. **Given** the player is setting up a game, **When** the phone changes orientation or viewport size, **Then** the setup view remains usable without clipped controls or unintended page-level horizontal scrolling.
4. **Given** a player looks up how to play on a phone, **When** they consult the project documentation, **Then** they can find the supported orientations, touch controls and known compatibility limits.
5. **Given** a player uses the game on a desktop device, **When** they use its existing keyboard, mouse or controller, **Then** those input methods remain available and behave as before.

### User Story 2 - Play a match with touchscreen controls (Priority: P1)

A player controls their character on a phone in portrait orientation with a touch-sensitive joystick at the bottom left and bomb/punch actions at the bottom right. If they rotate the phone to landscape, the existing horizontal control layout remains in use. The Punch action is shown only while that player has the Punch power.

**Why this priority**: Touchscreen match controls are the core difference between viewing the game on a phone and being able to play it there.

**Independent Test**: Join a match in portrait, steer from the bottom-left joystick and place bombs from the bottom-right action area; acquire Punch and verify its control appears and works. Rotate to landscape and verify the existing layout remains usable.

**Acceptance Scenarios**:

1. **Given** a phone player is in an active match, **When** they press and hold an on-screen direction, **Then** their character moves in that direction and stops when the touch is released or cancelled.
2. **Given** a phone player is in an active match, **When** they tap the bomb or punch control, **Then** the corresponding action is sent according to the existing game rules and action limits.
3. **Given** a player presses multiple on-screen controls at once, **When** those touches are supported by the device, **Then** movement and an action can be used together without one touch unexpectedly cancelling the other.
4. **Given** a match is active, **When** the player touches a gameplay control, **Then** that input does not accidentally scroll, zoom or select page text instead of playing.
5. **Given** the player holds a movement direction in the left zone and taps an action in the right zone, **When** the device supports simultaneous touches, **Then** both inputs are received without one cancelling the other.
6. **Given** a touch is interrupted by a phone call, browser focus change or pointer cancellation, **When** the player returns to the game, **Then** movement and action inputs are released and the game remains responsive.
7. **Given** a match is active in portrait orientation, **When** the view appears, **Then** the joystick is reachable at bottom left and bomb/power actions are reachable at bottom right without covering essential match information.
8. **Given** the phone changes to landscape during an active match, **When** the viewport changes, **Then** the existing horizontal control layout remains usable and held input is cleared during the transition.
9. **Given** the current player does not have the Punch power, **When** the mobile controls are shown, **Then** no Punch button is displayed; after the player acquires Punch, the button appears and can be used.
10. **Given** a player opens the game in Firefox for Android with touch input available, **When** they reach a match, **Then** the mobile controls are enabled without requiring a browser-specific setting or user-agent exception.

### User Story 3 - Review the round result and return to the lobby (Priority: P1)

When a round ends, players first see who won and a final scoreboard with each participant's standing and end-of-round stats. The scoreboard remains visible until the player chooses Return to Lobby; the lobby is not shown before that action.

**Why this priority**: Players need a clear ending and a deliberate way to move from the completed round into another game.

**Independent Test**: Complete or simulate a round ending, verify the winner/draw and all participant standings and stats remain visible, then use Return to Lobby and verify the lobby appears.

**Acceptance Scenarios**:

1. **Given** a round ends with a winner, **When** the result view appears, **Then** it names the winner and displays every participant's final standing and available end-of-round stats.
2. **Given** a round ends in a draw or a team/chaser mode outcome, **When** the result view appears, **Then** it describes that outcome accurately and displays the corresponding participants, teams or roles.
3. **Given** the server broadcasts the lobby state after the round ends, **When** the result view is still open, **Then** the result and scoreboard remain visible.
4. **Given** the result view is open, **When** a player activates Return to Lobby, **Then** the result view closes and the current lobby is shown.

### Edge Cases

- The phone uses a narrow viewport or a large text setting in either orientation: the arena, joystick zone, action zone and essential match information remain available without overlapping.
- A match starts or resumes in portrait: portrait controls are active; no orientation lock or rotate prompt is required.
- A phone changes between portrait and landscape while a control is held: the current layout adjusts and all held actions are released.
- The browser's address bar or on-screen keyboard changes the usable viewport while a menu or text field is open: the active controls remain visible and usable.
- A player rapidly taps an action or maintains a touch after moving a finger outside its control: inputs follow the game's action limits and are reliably released.
- Two or more touch controls are pressed simultaneously: each active touch remains associated with the intended control until released or cancelled.
- The browser loses focus, the page is backgrounded, or touch input is cancelled during movement: no gameplay input remains stuck when play resumes.
- The browser exposes touch points but reports a fine primary pointer: mobile controls remain enabled.
- A browser reports a fine primary pointer but also has touch points available: touch-capable controls remain enabled.
- The lobby message arrives immediately after the round-end message: the result screen remains until Return to Lobby is activated.
- A draw or mode-specific team/chaser outcome has no single winning player: the result title and standings still identify the correct outcome.
- The device has no touch input or does not meet the supported mobile-browser baseline: existing desktop input remains available and the page reports an actionable compatibility problem when it cannot be played.
- A phone player shares a room with desktop, terminal or bot players: all participants continue to see the same game state and result.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: A player MUST be able to open the existing game address in a supported mobile browser and reach the same game service without installing a separate application.
- **FR-002**: The phone interface MUST support the complete player-facing flow from room creation or joining through character choice, lobby use, available settings, match play and return to the lobby using touch input.
- **FR-003**: A touch-capable player MUST be able to play in portrait or landscape without an orientation lock or forced-rotation prompt; the mobile layout MUST adapt to the current orientation.
- **FR-004**: In portrait, the active match MUST place a touch-sensitive joystick at the bottom left and the action controls at the bottom right, with the arena and essential match information remaining visible. In landscape, it MUST retain the existing horizontal layout.
- **FR-005**: The left joystick zone MUST translate the player's touch direction into one of the game's four movement directions, provide visible feedback for the selected direction, and return to neutral when released, cancelled or when the page loses focus.
- **FR-006**: The right action zone MUST provide a bomb control. It MUST show the Punch control only while the player has the Punch power; each action MUST follow the existing game's action limits.
- **FR-007**: The touch zones MUST support simultaneous movement and action presses on devices that provide multi-touch, without one touch unexpectedly cancelling another.
- **FR-008**: Touch interactions with game controls MUST NOT cause unintended page scrolling, zooming, text selection or browser navigation during play.
- **FR-009**: When the page is backgrounded, loses focus or receives a cancelled touch, it MUST clear held gameplay input before accepting further play.
- **FR-010**: Text entry in room and player fields MUST work with the phone's standard on-screen keyboard, and the page MUST keep the active field and relevant actions usable while that keyboard is open.
- **FR-011**: Mobile play MUST preserve the existing match rules and multiplayer behavior; phone input MUST produce the same game actions and outcomes as equivalent supported inputs on other clients.
- **FR-012**: Existing desktop keyboard, mouse and controller play MUST remain available and unchanged when the game is used on non-mobile devices.
- **FR-013**: The project documentation MUST explain mobile-browser access, portrait and landscape layouts, conditional Punch visibility and known compatibility limits.
- **FR-014**: Mobile input detection MUST support touch-capable browsers, including Firefox for Android, without relying on a browser-name allowlist; touch support MUST remain enabled when any touch-capable pointer or touch points are available.
- **FR-015**: When a round ends, the web client MUST show the winner, draw or mode-specific outcome and a final scoreboard containing every participant's standing and available final stats before offering Return to Lobby.
- **FR-016**: The result view MUST remain visible when the server broadcasts the post-round lobby state and MUST NOT show the lobby until the player activates Return to Lobby.
- **FR-017**: The result view MUST provide an accessible Return to Lobby button operable by touch, keyboard and controller.

### Key Entities *(include if data involved)*

- **Touch control**: An on-screen control associated with a game action, including its active state and the touch currently operating it.
- **Mobile game view**: The current room, lobby, settings or match presentation arranged for the phone's usable screen area and orientation.
- **Touch input state**: The set of gameplay actions currently held by a player, which must be released when a touch ends or is cancelled.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: In 100% of tested phone-browser flows, a player can open the game, join or create a room, and reach a match using only touchscreen input.
- **SC-002**: In 100% of tested phone-browser match scenarios, a player can move, place a bomb and use the punch action without a physical keyboard, mouse or controller.
- **SC-003**: In 100% of tested portrait and landscape phone sizes, the appropriate control layout and essential match status remain visible without overlap or unintended page scrolling.
- **SC-004**: In 100% of tested touch releases, cancellations, focus losses and background/foreground transitions, no gameplay action remains held after the input ends.
- **SC-005**: In 100% of tested mixed-device matches, phone, desktop and terminal participants observe the same match outcomes.
- **SC-006**: A first-time phone player can identify the left joystick zone and the right bomb and punch controls, then complete each action without instructions from another player.
- **SC-007**: In 100% of tested orientation changes, portrait and landscape layouts remain usable and no held input persists across the transition.
- **SC-008**: In 100% of tested Firefox for Android and other touch-capable browser flows, the touch controls are detected and usable without browser-specific configuration.
- **SC-009**: In 100% of tested round endings, the correct outcome and every participant's final standings/stats remain visible through the lobby update, and the lobby stays hidden until Return to Lobby is activated.

## Assumptions


- Mobile support means playing through the existing game webpage in a phone browser; a native iOS or Android application is not required.
- Touchscreen controls are the primary phone gameplay input. A physical keyboard, mouse or controller may be used when available but is not required.
- Supported mobile browsers include current Firefox for Android and mainstream iOS/Android browsers that provide standard touch input and a viewport suitable for the page.
- Mobile matches support the phone's current orientation: portrait uses bottom-corner controls; landscape retains the existing horizontal layout. Orientation locking is not required.
- The left joystick zone selects one of the existing four movement directions; diagonal movement is not introduced.
- The scoreboard uses the round outcome and end-of-round participant stats already available; the feature adds no cumulative point or kill-count system.
- Existing hosting arrangements, room capacity, game rules and desktop behavior remain unchanged.
- The phone has a working network connection to the same game server used by other clients.