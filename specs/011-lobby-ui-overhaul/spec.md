# Feature Specification: Menu, Lobby, Customization and Scoreboard Reorganization

**Feature Branch**: `011-lobby-ui-overhaul`

**Created**: 2026-10-29

**Status**: Draft

**Input**: User description: "Reorganize things: main menu (Create new game / Join a game with room list / Options), tabbed Options (Customize, Sound, Controls), simplified Customize panel (remove gender Look, add Head/Face/Costume, move Randomize to an icon), remove the red focus border, redesign the Lobby (bots fill empty slots by default, small cross to kick/remove, + to add a bot, a configuration icon for room settings/password, leave-room icon, Ready for everyone + Start for owner only, remove the old Add/Remove bot and Change look buttons), and a reworked Scoreboard (ranked by survival order, showing only Player, Kills, Bombs placed)."

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Navigate from a clear main menu (Priority: P1)

A player opens the game and sees three top-level choices: start a brand-new game, browse and join an existing room, or open Options. Today there is no single entry screen — a player lands directly on room creation. Instead, the player should land on a menu that clearly separates "create" from "join" from "configure my experience."

**Why this priority**: This is the first screen every player sees; without it, every other reorganization (lobby, options) has no coherent entry point.

**Independent Test**: Launch the client fresh and verify three distinct actions are offered: Create new game, Join a game, Options — with no room-creation form shown until "Create new game" is chosen.

**Acceptance Scenarios**:

1. **Given** a player opens the client, **When** the main menu appears, **Then** it offers exactly three actions: "Create new game," "Join a game," and "Options."
2. **Given** the main menu, **When** the player selects "Create new game," **Then** the existing room-creation form (name, mode, board size, round time, capacity, password, friendly-fire) is shown.
3. **Given** the main menu, **When** the player selects "Join a game," **Then** a room list is shown with a button to refresh the list and a button to create a new room, replacing today's standalone directory panel as the destination of this action.
4. **Given** the room list, **When** the player presses refresh, **Then** the list re-queries available rooms and updates without navigating away from the list.
5. **Given** the main menu, **When** the player selects "Options," **Then** the tabbed options screen described in User Story 3 opens.

---

### User Story 2 - Manage a lobby with bot-filled slots (Priority: P1)

A player creates or joins a room and lands in the lobby. Empty seats are filled with bots automatically so the room feels ready to play immediately. The room owner can remove a bot or kick a player with one click, or add a bot back into an empty seat with one click. Room configuration (including the password) lives behind a single configuration icon, and leaving the room is a single icon — both in the lobby's header area rather than scattered action buttons at the bottom. Every player has one "Ready" action; only the owner additionally has "Start."

**Why this priority**: The lobby is where players spend time between matches; today's flat button row (Start, Add bot, Remove bot, Ready, Change look, Leave room) is cluttered and some of those actions either duplicate settings now reachable elsewhere (Change look) or are better expressed as inline per-slot controls (Add/Remove bot).

**Independent Test**: Create a room with capacity 4 and no other players; verify the 3 empty seats are immediately occupied by bots. Remove a bot via its per-slot control and verify the seat becomes empty with a "+" control. Click "+" and verify a bot fills the seat again. As owner, open the configuration icon and verify room settings and password are editable there and nowhere else in the lobby. Click the leave icon and verify the player exits the room.

**Acceptance Scenarios**:

1. **Given** a newly created or newly joined room with empty seats, **When** the lobby is displayed, **Then** every empty seat is automatically filled by a bot without any manual action.
2. **Given** a seat occupied by a bot or by another player, **When** the room owner clicks the small cross control on that seat, **Then** a bot is removed (seat becomes empty) or a player is kicked from the room (seat becomes empty), respectively.
3. **Given** a non-owner player, **When** they view the lobby, **Then** they do not see the cross control on any seat (they cannot kick or remove bots).
4. **Given** an empty seat, **When** any player views the lobby, **Then** the empty seat shows a "+" control that adds a bot to that seat when clicked.
5. **Given** the lobby, **When** the owner clicks the configuration icon in the lobby's top-right, **Then** a panel opens containing room name, mode, board size, round time, capacity, friendly-fire, and password controls — these controls no longer appear inline in the lobby body.
6. **Given** a non-owner player, **When** they view the lobby, **Then** the configuration icon is not shown (or is shown but does not grant edit access) and they cannot view or change the password from the lobby.
7. **Given** the lobby, **When** any player clicks the leave icon (located in the same header area as the configuration icon), **Then** they leave the room, replacing today's bottom-row "Leave room" button.
8. **Given** the lobby's action area, **When** it is displayed, **Then** it shows exactly two buttons in order: "Ready" (available to every player, including the owner) and "Start" (visible/usable only for the owner).
9. **Given** the lobby, **When** it is displayed, **Then** there are no "Add bot," "Remove bot," or "Change look" buttons anywhere in the lobby (their functions are covered by the per-slot controls and the Customize options tab, respectively).

---

### User Story 3 - Configure settings from tabbed Options (Priority: P2)

A player opens Options and finds three tabs: Customize, Sound, Controls. Today sound and controls exist as separate panels reachable from different places, with no unified Options entry and no tab for character customization.

**Why this priority**: Consolidating settings reduces navigation hunting and is a prerequisite for the Customize redesign in User Story 4.

**Independent Test**: Open Options from the main menu and from the in-game pause menu; verify both land on the same tabbed screen with Customize, Sound, and Controls tabs, and that switching tabs preserves unsaved changes made in another tab during the same session.

**Acceptance Scenarios**:

1. **Given** the Options screen is open, **When** it is displayed, **Then** exactly three tabs are shown: "Customize," "Sound," "Controls."
2. **Given** the Options screen, **When** the player selects the Sound tab, **Then** the existing sound controls (music/effects volume, mute, test sound) are shown.
3. **Given** the Options screen, **When** the player selects the Controls tab, **Then** the existing control bindings (key bindings, controller selection) are shown.
4. **Given** the Options screen, **When** the player selects the Customize tab, **Then** the character customization panel from User Story 4 is shown.
5. **Given** Options was opened from inside an active game (pause menu), **When** the player closes Options, **Then** they return to the paused game exactly as today's Sound/Controls panels already do.

---

### User Story 4 - Customize a character without the unneeded gender choice (Priority: P2)

A player opens the Customize tab to personalize their character's color, head, face, and costume. The previous "Look" (Male/Female) choice is removed as it added no value distinct from the other customization options. A new "Costume" category is added alongside the existing Head and Face categories. Randomizing the character's appearance is done via a small icon placed at the bottom-right corner of the character preview image, rather than a separate labeled button.

**Why this priority**: Depends on the Options tab structure (User Story 3) existing first; it is a focused simplification of one existing panel.

**Independent Test**: Open Customize and verify there is no gender/"Look" control; verify Head, Face, and Costume selectors are present; verify clicking the small icon at the bottom-right of the character preview randomizes color, head, face, and costume together.

**Acceptance Scenarios**:

1. **Given** the Customize tab, **When** it is displayed, **Then** there is no control for choosing "Male"/"Female" or any equivalent gender/"Look" selection.
2. **Given** the Customize tab, **When** it is displayed, **Then** it offers selectors for Head, Face, and Costume (Costume being a new category not previously present).
3. **Given** the Customize tab, **When** the player clicks the small randomize icon positioned at the bottom-right of the character preview image, **Then** the color, head, face, and costume are each set to a random valid value.
4. **Given** the Customize tab, **When** it is displayed, **Then** there is no separate full-size "Random" button outside the character preview image.
5. **Given** a character whose previously-saved appearance included a gender/"Look" value from before this change, **When** that appearance is loaded, **Then** it is displayed without error using only color, head, face, and a default costume.

---

### User Story 5 - See a clear, ranked scoreboard after a match (Priority: P2)

At the end of a match, the player sees a scoreboard listing players ordered by how well they did — the last player still alive first, the next-to-last-eliminated player second, and so on — showing only the player's name, how many kills they got, and how many bombs they placed. Today the table is ordered by lobby slot and shows Result/Bombs/Range/Speed/Powers columns instead.

**Why this priority**: Improves post-match clarity; independent of the menu/lobby/customize changes but still a discrete, user-visible reorganization.

**Independent Test**: Finish a 4-player match with a known elimination order; verify the scoreboard lists the last-alive player first, then players in reverse elimination order, and verify each row shows only name, kills, and bombs placed.

**Acceptance Scenarios**:

1. **Given** a finished match, **When** the scoreboard is displayed, **Then** players are listed in order of survival: the last player standing (or the winner) first, followed by players in the reverse order they were eliminated (most recently eliminated appears earlier than someone eliminated earlier in the match).
2. **Given** the scoreboard, **When** it is displayed, **Then** each row shows exactly three fields: the player's name, their number of kills, and their number of bombs placed — no other columns are shown.
3. **Given** a mode without eliminations (e.g., a draw or a mode where no player is ever removed), **When** the match ends, **Then** all players are still listed with a well-defined, stable order and their kills/bombs placed are still shown.
4. **Given** two players tied on survival (eliminated in the same tick, or both surviving to a draw), **When** the scoreboard is displayed, **Then** a consistent tie-breaking order is applied (e.g., by kills, then by original slot) so the scoreboard order is deterministic.

---

### User Story 6 - No distracting red outline on focused elements (Priority: P3)

A player tabbing through or otherwise focusing interactive elements (buttons, inputs, selects) no longer sees a red focus outline. Today a bright red outline (`#d43b32`) appears via `:focus-visible` styling.

**Why this priority**: Purely cosmetic; lowest priority but trivial and independent of all other stories.

**Independent Test**: Tab through the main menu, lobby, and options screens and verify no element shows a red outline when focused, while some non-red, visible focus indication still exists for keyboard/controller accessibility.

**Acceptance Scenarios**:

1. **Given** any interactive element (button, input, select, radio) in the client, **When** it receives keyboard or controller focus, **Then** no red outline is shown.
2. **Given** any interactive element receives focus, **When** it is displayed, **Then** a visible, non-red focus indicator is still present (focus visibility is not removed entirely, only the red color).

---

### Edge Cases

- What happens if the owner removes a bot or kicks a player from a slot that is also the slot being used for the configuration panel or another in-progress action? The slot update must be reflected immediately in the configuration panel and seat list without requiring a manual refresh.
- What happens if all seats are filled by bots and a human player later joins? One bot slot must be freed (replaced by the joining human) rather than rejecting the join, consistent with existing capacity rules.
- What happens if a non-owner tries to directly invoke an owner-only action (kick, remove bot, open configuration panel, Start) through a stale UI state (e.g., ownership changed while their panel was open)? The action must be rejected server-side and the UI must reconcile to the current ownership state.
- What happens when the owner disconnects? Existing ownership transfer behavior is unchanged by this feature; the new configuration icon and kick controls simply follow whoever currently holds ownership.
- What happens to the Join-a-game room list when zero rooms exist? It shows an empty state plus the refresh and new-room controls, consistent with today's directory panel behavior.
- What happens to a player's in-progress customization choices if they switch Options tabs without an explicit save? Changes made in the Customize tab are preserved when switching to Sound or Controls and back, consistent with how the existing Sound/Controls panels already retain changes.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: The client MUST present a main menu with exactly three actions: Create new game, Join a game, Options.
- **FR-002**: Selecting "Join a game" MUST open a room list screen offering a refresh action and a create-new-room action.
- **FR-003**: The Options screen MUST be organized into exactly three tabs: Customize, Sound, Controls, each reachable from both the main menu and any in-game pause menu that currently links to Sound/Controls.
- **FR-004**: The Customize tab MUST NOT include a gender/"Look" (Male/Female) selector.
- **FR-005**: The Customize tab MUST include selectors for Head, Face, and Costume.
- **FR-006**: The Customize tab MUST provide a randomize action as a small icon overlaid at the bottom-right corner of the character preview image, and MUST NOT provide a separate full-size "Random" button.
- **FR-007**: The client MUST NOT display a red outline on any focused interactive element, while still providing a visible non-red focus indicator.
- **FR-008**: The lobby MUST automatically fill every empty seat with a bot when a seat becomes empty (room creation, a player leaving without being replaced, or a bot being removed and the seat not otherwise re-filled) — except for seats intentionally left empty by an explicit removal action, which are shown with a "+" control instead.
- **FR-009**: Every occupied seat (bot or player) MUST display a small control (a cross) that the room owner can use to remove that bot or kick that player; this control MUST NOT be shown to, or usable by, non-owner players.
- **FR-010**: Every empty seat MUST display a "+" control, usable by any player, that adds a bot to that seat.
- **FR-011**: The lobby MUST provide a configuration icon, visible to the owner, that opens a panel containing the room's name, mode, board size, round time, capacity, friendly-fire setting, and password controls; these controls MUST NOT appear directly in the lobby body outside that panel.
- **FR-012**: Non-owner players MUST NOT be able to view or change room configuration or the password.
- **FR-013**: The lobby MUST provide a leave-room icon placed in the same header area as the configuration icon.
- **FR-014**: The lobby MUST NOT include "Add bot," "Remove bot," or "Change look" buttons.
- **FR-015**: The lobby's action area MUST show a "Ready" action usable by every player and a "Start" action usable only by the owner, in that order.
- **FR-016**: The server MUST reject kick, bot-removal, configuration-change, and Start actions from any non-owner, regardless of what the requesting client's UI displayed.
- **FR-017**: The end-of-match scoreboard MUST order players by survival: the last player alive (or match winner) first, then remaining players in reverse order of elimination (most recently eliminated listed before earlier-eliminated players), with a deterministic tie-break when two players share the same elimination point.
- **FR-018**: The end-of-match scoreboard MUST show only each player's name, number of kills, and number of bombs placed, and MUST NOT show any other column.
- **FR-019**: The game MUST track, per player per match, a count of kills and a count of bombs placed, sufficient to populate the scoreboard in FR-018.

### Key Entities

- **Room Configuration**: name, game mode, board size, round time, max capacity, friendly-fire setting, password — owner-editable, now grouped behind a single configuration panel.
- **Player Slot**: a seat in a room; holds either a human player, a bot, or is empty; carries the per-slot kick/remove-bot control and the empty-slot add-bot control.
- **Character Appearance**: color, head, face, costume — no longer includes a gender/"Look" value.
- **Match Scoreboard Entry**: player name, kills, bombs placed, and the player's survival rank used for ordering (last-alive first).

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: A new player can go from launching the client to being in a full (bot-filled) lobby in 2 actions or fewer (e.g., "Create new game" then confirm), with zero required manual bot-adding steps.
- **SC-002**: 100% of owner-only actions (kick, remove bot, open configuration, Start) are inaccessible to non-owner players in the UI and are rejected if attempted anyway.
- **SC-003**: The number of always-visible action buttons in the lobby's main action area is reduced from six (Start, Add bot, Remove bot, Ready, Change look, Leave room) to two (Ready, Start), with the remaining functions moved to per-slot controls, a configuration panel, or the Customize options tab.
- **SC-004**: 100% of post-match scoreboards list the eventual winner or last-survivor in the first row, verified across Normal, Team, and Chaser modes.
- **SC-005**: Zero interactive elements show a red focus outline across the main menu, lobby, and options screens, verified by tabbing through each screen.

## Assumptions

- "Kills" is a new per-player, per-match statistic to surface on the scoreboard; it is derived from existing elimination/blast-attribution data already needed to determine match results, not a new gameplay rule.
- "Bombs placed" is a per-player, per-match count of bomb-placement actions, independent of whether those bombs detonated, were defused, or hit anything.
- Default bot fill applies to every room at creation and whenever a seat becomes empty during the lobby phase (pre-start); it does not apply mid-match (a mid-match departure continues to be handled by existing bot-takeover behavior, unchanged by this feature).
- "Costume" is a new appearance category distinct from the existing Head and Face categories and from the removed gender/"Look" category; the specific set of costume options is an implementation/content decision, not a scope constraint of this spec.
- The in-game pause menu's existing Sound and Controls entry points are updated to open the new tabbed Options screen (defaulting to the Sound or Controls tab as appropriate) rather than being duplicated as separate panels.
- This feature is scoped to the web client UI and its supporting server-side authorization (owner checks) and statistics tracking; it does not change core bomb/blast/movement rules beyond what is needed to count kills and bombs placed.
