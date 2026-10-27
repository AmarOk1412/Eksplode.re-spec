# Feature Specification: Web Client and Project Documentation

**Feature Branch**: `002-web-client`

**Created**: 2026-10-22

**Status**: Draft

**Input**: User description: "Graphics will be a web page. Ideally we can launch a server (warning of CORS) and join the webpage to create a new player on this instance (or via a cli option to join in CLI mode). Note: Add documentation and README to explain the architecture of the project and how everything is working"

**Relationship to 001**: This feature replaces the native graphical window of User Story 4 in `001-bomberman-multiplayer`. The isometric look with coloured cubic players is kept, but it is delivered as a web page served by the game server. The terminal version, bots and network rules of 001 stay as they are.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Join a running match from a web browser (Priority: P1)

A person starts the game server on one machine. The server prints a web address. Anyone who opens that address in a browser sees the lobby of that same server, picks a name, and takes a free slot as a new player. When the match starts they play it in the browser with the keyboard, seeing the arena in an isometric view with each player drawn as a coloured cube.

**Why this priority**: This is the whole point of the feature: a graphical client that needs no installation and plays on the same instance as everybody else.

**Independent Test**: Start the server, open its address in a browser, join, start a match with at least one bot, and play a round to its result using only the browser.

**Acceptance Scenarios**:

1. **Given** a running server, **When** a user opens its web address in a browser, **Then** the page loads without any installation and shows the lobby with the current slots.
2. **Given** the lobby page, **When** the user enters a valid name and joins, **Then** they get a free slot and see it marked as theirs; **When** all 4 slots are taken, **Then** the page says the match is full.
3. **Given** a started match, **When** the browser shows it, **Then** the arena is drawn isometrically with walls, blocks, bombs, flames, power-ups, and each player as a distinct coloured cube.
4. **Given** a match in the browser, **When** the user presses the movement keys and the bomb key, **Then** their cube moves and drops bombs under the same rules as in the terminal version.
5. **Given** a round that ends, **When** the result is known, **Then** the page announces the winner or a draw and returns to the lobby for a rematch.
6. **Given** a joined browser player, **When** the connection to the server is lost, **Then** the page says so clearly and lets the user try to reconnect.

---

### User Story 2 - Play browser, terminal and bot players together (Priority: P2)

On one server instance, browser players, terminal players (joining by address) and bots can share a match, for up to 4 participants in total. Everybody sees the same game and the same result, whichever client they use.

**Why this priority**: It proves the browser is a real peer of the existing clients and not a separate game.

**Independent Test**: Host a match, join one browser and one terminal client, add a bot, play a round, and check that all see the same outcome.

**Acceptance Scenarios**:

1. **Given** a server with a browser player and a terminal player in the lobby, **When** the match starts, **Then** both see the same arena and starting positions.
2. **Given** a mixed match, **When** any participant acts, **Then** all other participants see the result, and all agree on who was eliminated and who won.
3. **Given** a browser player closes the page during a match, **When** the server notices, **Then** the others are told and a bot takes over the slot, as for any other disconnected player.
4. **Given** a full match (4 participants), **When** another browser or terminal client tries to join, **Then** it is refused with a clear message.

---

### User Story 3 - Launch and join from the command line (Priority: P3)

A person can start the game server from the command line with a single command, choosing the port and how many bots to add. The server tells them the address to share. Another person can join the same instance from the terminal, using an address given on the command line, instead of using a browser.

**Why this priority**: This is the entry point for hosting, and keeps terminal-only users first-class.

**Independent Test**: Run the server command, read the printed address, join once from a browser and once from a terminal command, and start a match.

**Acceptance Scenarios**:

1. **Given** the server command with default options, **When** it starts, **Then** it prints the web address, the address for terminal clients, and how to stop it.
2. **Given** the command with a port option, **When** the port is already in use, **Then** it stops with a clear message and a non-zero exit status.
3. **Given** a terminal join command with the server address, **When** the server is reachable, **Then** the user lands in the same lobby as the browser players.
4. **Given** the terminal lobby shown to the host, **When** the host presses Enter to start, **Then** the match starts reliably (this fixes the reported problem where Enter had no effect).

---

### User Story 4 - Understand the project from its documentation (Priority: P4)

A new reader opens the README and the documentation and learns what the project is, how it is built and organised, how a match flows from key press to screen, how to run every mode, and how to run the tests. A diagram shows the main parts and how the clients, server and game rules relate.

**Why this priority**: The user asked explicitly for documentation of the architecture and how everything works. It is lower priority only because it can be completed last, once behaviour is settled.

**Independent Test**: A person who has never seen the project follows the README alone to build it, start a server, join from a browser and a terminal, and run the tests.

**Acceptance Scenarios**:

1. **Given** the README, **When** a new user follows the quick start, **Then** they reach a running match in the browser without outside help.
2. **Given** the architecture documentation, **When** a contributor looks up where a rule, a bot decision, a network message or a screen is implemented, **Then** the documentation points to the right part and explains how it fits.
3. **Given** the documentation, **When** a reader looks for the safety notes, **Then** they find the explanation of the cross-origin and open-network warnings and how to host safely.
4. **Given** a change to behaviour or options, **When** the project is reviewed, **Then** the documentation affected is updated in the same change.

---

### Edge Cases

- The page is opened from another site or origin than the server: the server must refuse cross-origin use by default and say so; allowing other origins is an explicit option that prints a warning.
- Two browser tabs from the same machine join: each is a separate player and counts toward the 4 slots.
- A browser tab goes to the background and its updates slow down: the player must not be removed unfairly or see an inconsistent state when it comes back.
- A user joins with an invalid name or a name already in use: they get a clear message and can correct it.
- The server is stopped while browsers are connected: pages show that the server is gone.
- A browser loses focus while keys are held: held movement stops instead of continuing.
- A browser with a very small window: the arena scales to fit or the page asks for a larger window.
- A browser without the needed capabilities: the page says what is missing instead of showing a blank screen.
- Someone sends malformed or oversized data to the server from a web page: it is rejected without affecting the match.
- The server is started on a machine whose address is not obvious (several network interfaces): it prints every usable address.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: The server MUST deliver a web page that works in current mainstream desktop browsers with no installation and no separate build step for the person running the game.
- **FR-002**: A browser user MUST be able to open the server's address, enter a name, join the lobby as a new player and leave again.
- **FR-003**: A browser player MUST see the match in an isometric view with walls, blocks, bombs, flames, power-ups and each of the 4 players as a distinct coloured cube, identifiable by colour and number.
- **FR-004**: A browser player MUST be able to control their player with the keyboard (move in 4 directions, place a bomb) with the same rules and speed as other clients.
- **FR-005**: Browser, terminal and bot participants MUST be able to share one match, up to 4 participants in total, and all MUST see the same game state and result.
- **FR-006**: The server MUST be started with one command that accepts a port and a number of bots, and MUST print the web address, the terminal join address and how to stop it.
- **FR-007**: A terminal client MUST be able to join the same server by giving its address on the command line.
- **FR-008**: The host in the terminal lobby MUST be able to start the match with the Enter key, and a failure to start MUST be shown to the user.
- **FR-009**: The server MUST refuse requests coming from other web origins by default; allowing them MUST be an explicit option that prints a clear warning about the risks at start-up and is documented.
- **FR-010**: When the server is reachable beyond the local machine, it MUST warn that the game has no encryption or authentication and say how to restrict it to the local machine.
- **FR-011**: A browser player who disconnects, closes the page or goes silent MUST be handled like any other disconnected player: others are told and a bot takes the slot.
- **FR-012**: The page MUST show connection problems (refused, full, lost, server gone) in plain language and offer a way to retry.
- **FR-013**: The server MUST reject malformed, oversized or out-of-turn messages from web clients without affecting the match or other players.
- **FR-014**: The game rules MUST be the same for all clients: a browser client MUST NOT contain or decide game rules, only show state and send intentions.
- **FR-015**: The project MUST include a README and documentation that explain the purpose, the architecture (with a diagram), the flow of a match, how to run every mode, how to run the tests, the safety notes, and where each part lives in the code.
- **FR-016**: All new behaviour, including the server, the browser-facing messages, the terminal join option and the lobby start fix, MUST be covered by automated tests, and each acceptance scenario of this specification MUST map to at least one automated test or documented manual check.
- **FR-017**: Delivery order MUST be: fix the terminal lobby start problem, then the server that delivers the page and accepts web connections, then the terminal join option, then the documentation.

### Key Entities

- **Server instance**: One running game server with one lobby and at most one running match; it serves the page and accepts both browser and terminal participants.
- **Web participant**: A person playing through a browser page; occupies one slot, has a name, and can be disconnected at any time.
- **Terminal participant**: A person playing through the terminal client on the same instance.
- **Page**: The web page delivered by the server; shows the lobby, the isometric match and the results, and sends the player's intentions.
- **Connection notice**: A message shown to the user about joining, refusal, being full, disconnection or server shutdown.
- **Documentation set**: README plus architecture and usage documents, including a diagram of the parts and how they talk to each other.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: A new user can go from starting the server to playing in a browser in under 2 minutes by following the README, without installing anything besides the game itself.
- **SC-002**: In a match with one browser, one terminal and two bot participants, 100% of tested rounds end with all clients reporting the same result.
- **SC-003**: 95% of a browser player's key actions are visible on their screen within 200 ms on a local network.
- **SC-004**: A first-time viewer can tell the 4 players apart by colour and number, and tell bombs, flames, walls and blocks apart, without guidance.
- **SC-005**: The game stays visually smooth (no visible stutter) during a full 4-participant round with chain explosions in the browser.
- **SC-006**: In 100% of tested cases, a browser player closing the page leaves the match running for the others.
- **SC-007**: 100% of tested cross-origin requests are refused under default settings, and enabling them always prints the warning.
- **SC-008**: Starting a match with Enter in the terminal lobby succeeds in 100% of tested attempts.
- **SC-009**: In a review by someone new to the project, at least 4 of 5 architecture questions (for example where rules live, how a message travels from key press to screen, how bots plug in) are answered correctly using only the documentation.
- **SC-010**: 100% of the acceptance scenarios in this specification are covered by an automated test or a listed manual check that passes.

## Assumptions

- The server is used on a trusted local network or on the same machine; there are no accounts, no encryption and no internet-wide discovery, as in the previous feature.
- The web page is served by the same server that hosts the match, so by default the page and the game share one origin and need no special cross-origin set-up.
- Keyboard is the only input in the browser; touch screens and gamepads are out of scope.
- One match runs at a time per server instance, with at most 4 participants of any kind.
- Sound, music, accounts, persistent statistics and spectators are out of scope.
- The native graphical window from `001-bomberman-multiplayer` (User Story 4) is replaced by this web client and will not be built.
- Current versions of the major desktop browsers are supported; very old browsers are not.
- Documentation is written in English and kept in the repository next to the code.
- Existing terminal play, bots and network rules from `001-bomberman-multiplayer` keep working unchanged.
