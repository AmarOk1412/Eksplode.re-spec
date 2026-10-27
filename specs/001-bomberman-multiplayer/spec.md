# Feature Specification: Bomberman Multiplayer Game

**Feature Branch**: `001-bomberman-multiplayer`

**Created**: 2026-10-22

**Status**: Draft

**Input**: User description: "Create a bomberman, with a CLI and graphical version with bots and with 4 players (can be remote), with tests. The graphics will be isometric with cubic players."

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Play a local match in the terminal (Priority: P1)

A player launches the terminal version and plays a complete Bomberman round on a grid arena: moving, placing bombs, destroying soft blocks, collecting power-ups, and trying to be the last one standing. This is the smallest playable slice and establishes the full ruleset.

**Why this priority**: The ruleset is the foundation for bots, remote play, and the graphical version. Without a correct, playable game nothing else has value.

**Independent Test**: Start a match from the terminal, move a player, place a bomb, observe the explosion destroy a soft block and eliminate a player, and see a round result announced.

**Acceptance Scenarios**:

1. **Given** a new match, **When** it starts, **Then** the arena shows fixed walls, destructible blocks, and each participant in a distinct corner start position with a clear starting area.
2. **Given** a player next to an open tile, **When** the player moves toward it, **Then** the player occupies that tile; **When** moving toward a wall, block, or bomb, **Then** the player does not move.
3. **Given** a player who places a bomb, **When** the fuse time elapses, **Then** the bomb explodes in four directions up to its blast range, stopping at walls and destroying the first destructible block hit.
4. **Given** a bomb whose blast reaches another bomb, **When** the first explodes, **Then** the second detonates immediately (chain reaction).
5. **Given** a player caught in a blast, **When** the explosion occurs, **Then** that player is eliminated.
6. **Given** a destroyed block that hid a power-up, **When** a player walks onto it, **Then** the player's capabilities improve (more bombs, longer range, or faster movement).
7. **Given** only one player remains alive, **When** the last opponent is eliminated, **Then** the round ends and that player is declared the winner; **Given** all remaining players are eliminated at the same moment, **Then** the round is declared a draw.

---

### User Story 2 - Play against computer-controlled bots (Priority: P2)

A player starts a match and fills the remaining slots (up to 3 others) with bots. Bots move, avoid explosions, place bombs sensibly, and pursue opponents and blocks, so a single person can enjoy a full four-participant game.

**Why this priority**: Bots make the game playable alone and give automated, repeatable opponents for testing the rules.

**Independent Test**: Start a match with one human and three bots, or with four bots only, and verify that bots act on their own, avoid obvious self-destruction, and the round reaches a result without human input from the bots.

**Acceptance Scenarios**:

1. **Given** a match setup, **When** the player assigns a slot to a bot, **Then** that slot acts autonomously for the whole round.
2. **Given** a bot standing within the blast zone of a pending bomb with a safe path available, **When** time passes, **Then** the bot moves out of the zone before detonation.
3. **Given** a bot that places a bomb, **When** it does so, **Then** it has an escape route available (it does not knowingly trap itself).
4. **Given** a match with four bots only, **When** it runs, **Then** it ends with a winner or a draw within a bounded time.
5. **Given** a bot, **When** it decides on an action, **Then** it uses only information visible to a human participant and the same actions available to them.

---

### User Story 3 - Play with up to 4 players over a network (Priority: P3)

One player hosts a match and up to three others join from other machines. Any slot not taken by a remote human may be filled by a bot. All participants see the same game progress and can play with comfortable responsiveness.

**Why this priority**: Remote multiplayer is the headline feature but depends on a correct ruleset and the participant/bot model already being in place.

**Independent Test**: Host a match, join with a second instance (and up to three), play a round, and verify that all participants see the same outcome.

**Acceptance Scenarios**:

1. **Given** a host with an open match, **When** others join using the host's address, **Then** each is assigned a free slot and sees the shared lobby.
2. **Given** a match with 2 to 4 participants (human or bot), **When** the host starts it, **Then** all participants see the same arena and starting positions.
3. **Given** a running match, **When** any participant acts, **Then** every participant sees the resulting state, and all agree on who was eliminated and who won.
4. **Given** a remote player disconnects mid-round, **When** the connection is lost, **Then** the other participants are notified and the match continues, with the departed player's character removed or taken over by a bot.
5. **Given** a participant sending invalid or out-of-turn actions, **When** received by the host, **Then** they are rejected without affecting other participants.
6. **Given** a full match (4 participants), **When** a fifth participant tries to join, **Then** the join is refused with a clear message.

---

### User Story 4 - Play with the isometric graphical version (Priority: P4)

A player launches the graphical version and sees the arena in an isometric view, with each player depicted as a distinctly coloured cube. They can play local matches with bots and remote matches exactly as in the terminal version.

**Why this priority**: It is the richest presentation, but it is a different view of an already complete game.

**Independent Test**: Launch the graphical version, play a round with bots, and verify that the arena is shown isometrically, players appear as coloured cubes, and all game events (bombs, explosions, power-ups, elimination) are visible.

**Acceptance Scenarios**:

1. **Given** a match, **When** it is displayed, **Then** the arena is drawn from an isometric viewpoint with walls, blocks, bombs, flames, and power-ups visually distinguishable.
2. **Given** four participants, **When** they appear, **Then** each is a cube of a distinct colour, consistent between the lobby, the arena, and the result screen.
3. **Given** a player or cube hidden behind taller scenery, **When** drawn, **Then** it remains identifiable (correct depth ordering and visibility).
4. **Given** any rule outcome in the graphical version, **When** compared with the terminal version for the same sequence of actions, **Then** the outcome is identical.
5. **Given** the graphical version, **When** a player uses the keyboard, **Then** movement and bomb placement work with documented controls.

---

### Edge Cases

- A player places a bomb and steps off it, then returns: the bomb blocks movement once the owner has left its tile.
- Two bombs or flames overlap on a tile in the same moment: the tile is simply "in blast"; no duplicate eliminations or errors.
- A power-up is revealed in the blast of another bomb: it is destroyed by the flames.
- A player attempts to place a bomb on a tile that already has one, or above their bomb limit: the action is ignored.
- Players move into the same tile at the same moment: only one succeeds, resolved by a deterministic rule.
- A round lasts too long without elimination: the round ends after a time limit, with a deterministic outcome (e.g., a draw).
- A match is started with fewer than 4 participants: the arena is still fair, and unused corners are empty.
- The host leaves during a match: remaining participants are told the match ended.
- A connection is slow or briefly interrupted: the game recovers without the participants seeing inconsistent results.
- The terminal is too small for the arena: the user is told the minimum size needed.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: The game MUST provide a rectangular arena with indestructible walls, destructible blocks, and starting positions for up to 4 participants, each with a guaranteed clear starting area.
- **FR-002**: Participants MUST be able to move one tile at a time in four directions, and be blocked by walls, blocks, and bombs.
- **FR-003**: Participants MUST be able to place bombs that explode after a fixed fuse time, with blast range in four directions, stopped by walls and destroying the first destructible block.
- **FR-004**: Bombs reached by a blast MUST detonate immediately (chain reactions).
- **FR-005**: A participant caught in a blast MUST be eliminated.
- **FR-006**: Destroyed blocks MUST sometimes reveal power-ups that increase bomb count, blast range, or movement speed when collected.
- **FR-007**: A round MUST end when one or zero participants remain, or when a time limit is reached, and the result (winner or draw) MUST be announced.
- **FR-008**: The game MUST support up to 4 participants per match, each slot being a local human, a remote human, or a bot.
- **FR-009**: Bots MUST act autonomously, avoid blast zones, avoid trapping themselves when placing bombs, and use only information and actions available to humans.
- **FR-010**: A terminal version MUST be provided offering the complete ruleset, local play, bot opponents, and remote play.
- **FR-011**: A graphical version MUST be provided offering the same ruleset, local play, bot opponents, and remote play.
- **FR-012**: The graphical version MUST render the arena isometrically and depict each player as a distinctly coloured cube.
- **FR-013**: A player MUST be able to host a match and others MUST be able to join using the host's address; a match MUST refuse participants beyond 4.
- **FR-014**: In a remote match, all participants MUST see the same game progress and outcome; the host's view of the game is authoritative.
- **FR-015**: The game MUST reject invalid or malformed actions from any participant without disrupting the match.
- **FR-016**: The game MUST handle a remote player disconnecting by informing others and continuing the match.
- **FR-017**: Given the same starting conditions and the same sequence of actions, the game MUST produce identical outcomes (reproducible matches).
- **FR-018**: Controls and how to host, join, and add bots MUST be documented for both versions.
- **FR-019**: All game rules, bot behaviour, and remote-play message handling MUST be covered by automated tests, and every acceptance scenario in this specification MUST map to at least one automated test.
- **FR-020**: Features MUST be delivered in this order: terminal local play, then local bots, then remote multiplayer, then the graphical version.

### Key Entities

- **Match**: A session of one or more rounds with a set of participants, and the overall results.
- **Round**: One play-through of an arena from start to winner or draw.
- **Arena**: The grid of walls, destructible blocks, and open tiles, plus items on it.
- **Participant**: One of up to 4 slots, controlled by a local human, remote human, or bot; has a colour, position, alive/eliminated state, bomb capacity, blast range, and speed.
- **Bomb**: A placed explosive with an owner, location, and remaining fuse time.
- **Blast**: The transient flames produced by a bomb across affected tiles.
- **Power-up**: A collectible on a tile that improves a participant's capability.
- **Action**: A participant's intent for a moment in the game: move in a direction, place a bomb, or do nothing.
- **Lobby**: The pre-round state where participants join, choose slots, and bots are added.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: A new player can start and play a local terminal round within 1 minute of launching, using only the documented controls.
- **SC-002**: 100% of the acceptance scenarios in this specification are covered by automated tests that pass.
- **SC-003**: Across 100 matches of four bots only, 100% end with a winner or a draw within the time limit, with no stalls or crashes.
- **SC-004**: In scripted scenarios, bots escape an imminent blast when a safe path exists in at least 95% of cases.
- **SC-005**: Two runs with identical starting conditions and actions produce identical results in 100% of cases.
- **SC-006**: In a 4-participant remote match on a typical home network, participants see each other's actions within 200 ms for 95% of actions, and all participants agree on the final result in 100% of rounds.
- **SC-007**: A remote participant disconnecting does not stop the match for the remaining participants in 100% of tested cases.
- **SC-008**: In the graphical version, a first-time viewer can identify each of the 4 players by cube colour and tell bombs, flames, walls, and blocks apart without guidance.
- **SC-009**: The graphical version remains smooth for players (no visible stutter) during a full 4-player round with bombs and chain explosions.

## Assumptions

- Classic Bomberman rules apply: grid arena, last participant standing wins, bombs with fuse timers and cross-shaped blasts.
- Rounds are single-arena with a fixed layout style; blocks are randomly distributed but starting areas are always clear.
- "Remote" means play over a network, with one player hosting and others joining by address; no matchmaking server, accounts, or internet-wide discovery are in scope.
- Remote play targets a local network or direct connection; no network encryption or authentication is required for this version beyond rejecting invalid actions.
- Keyboard is the only input method; gamepad support is out of scope.
- Bots have a single difficulty level in this version.
- Sound, music, and persistent statistics are out of scope.
- The primary target platform is Linux; the terminal version needs a terminal of a minimum size that is reported if unmet.
- Delivery follows the sequence in FR-020, and each stage is a usable product on its own.
