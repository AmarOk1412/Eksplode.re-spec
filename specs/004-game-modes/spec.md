# Feature Specification: Game Modes and Room Settings

**Feature Branch**: `004-game-modes`

**Created**: 2026-10-23

**Status**: Draft

**Input**: User description: "The game supports several modes that the owner of the room can set (also it can set time and size of the board; the current default size is the minimal size). Start with 3 modes: Normal, like the current implemented mode, but during the last 60 seconds blocks fall in a pattern on the board until all players except the winner are dead. Team: 2 vs 2 (more players in a room may come later). Chaser: a few walls, but less; bonuses appear randomly on the map; 3 players get no bombs and can only run; the chaser starts in the center with 2 bombs and 3 hearts and his role is to kill the other people." Planning additions: "Add a room creation menu first where the owner can configure the room (it will be shared on the network in a future specification); implement the end pattern; implement the other settings; implement the new modes."

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Create and configure a room before playing (Priority: P1)

Before anybody plays, the person who opens a room sees a room creation menu: a room name, the game mode (Normal, Team or Chaser), the board size (small, medium or large) and the round time (2, 3, 5 or 10 minutes), plus the friendly-fire switch that only matters in Team mode. The defaults are the game as it is today: Normal mode, the small board (the smallest the game allows, 15 x 13) and 3 minutes. When the owner confirms, the room opens and everybody who joins afterwards sees the room name, the mode and the settings in the lobby. The owner can still change the settings in the lobby until the match starts; nobody else can, and nobody can during a match. The room is described by one self-contained set of settings, so that a later feature can publish rooms on the network; this feature only has one room per server.

**Why this priority**: Every mode and setting below needs a place where it is chosen and shown, and with only the Normal mode and the default settings the game still plays exactly as today, so this is a complete first increment.

**Independent Test**: Start a server, open the page as the first person, see the creation menu, pick a medium board and 5 minutes, confirm, and check that the lobby and a second person who joins show the same settings; try to change them from the second person and during a match and check that both are refused.

**Acceptance Scenarios**:

1. **Given** a server with no person in its room, **When** the first person joins from the page, **Then** the room creation menu is shown first with the defaults (Normal, small board, 3 minutes), and the person becomes the owner when they confirm it.
2. **Given** a room that is open, **When** another person joins, **Then** they go straight to the lobby, which shows the room name, the mode, the board size, the time and (in Team mode) the friendly-fire setting, and they cannot change them.
3. **Given** the owner in the lobby, **When** they change the mode, the board size, the time or the room name, **Then** everybody in the lobby sees the new value at once and the next match uses it.
4. **Given** a started match, **When** anybody tries to change a setting, **Then** it is refused with a clear message and nothing changes; **When** the match ends and the lobby returns, **Then** the same settings are still in place for the rematch.
5. **Given** a mode that needs four participants (Team, Chaser) and fewer than four in the lobby, **When** the owner tries to start, **Then** the start is refused with a message that says how many are missing and that bots can fill the slots.
6. **Given** a medium or large board, **When** a match starts, **Then** the board has that size, the four players start in its corners, the arena is still fair and fully reachable, and the page and the terminal fit it (the terminal tells the user when it is too small).
7. **Given** the terminal version, **When** a person hosts a room, **Then** they can set the same room name, mode, size, time and friendly-fire choices (by options and on a setup screen before the lobby), and terminal participants see the same settings in the lobby.
8. **Given** the last person leaves the lobby, **When** the next person joins, **Then** the room is created anew with the creation menu and the defaults.

---

### User Story 2 - The end pattern in Normal mode (Priority: P2)

In Normal mode the round no longer ends in a draw when the time runs out. During the last 60 seconds of the round time, blocks start to fall on the board in a fixed pattern that closes in from the outside to the middle, until only one player is left. Each block is announced on its square a moment before it lands, so everybody can run; a block that lands on a player eliminates them. The pattern is fast enough that the round always ends: it covers every free square about one minute after it starts, however large the board is.

**Why this priority**: It turns the existing round into one that always has a winner and gives the new time setting its meaning.

**Independent Test**: Start a Normal match with 2 minutes and one bot that stands still, wait for the last 60 seconds, and check that warnings and falling blocks appear from the edge inwards, that a player on a falling square is eliminated, and that the round ends with a winner or a draw only when nobody remains.

**Acceptance Scenarios**:

1. **Given** a Normal match, **When** the remaining time reaches 60 seconds, **Then** the end pattern begins: squares at the outer edge are marked, and a short moment later a block lands on each marked square, spiralling inwards in a fixed order.
2. **Given** a marked square, **When** its block lands, **Then** any player on it is eliminated, a bomb or a power on it is destroyed (a bomb does not explode), and the square becomes a block that is not destructible by explosions.
3. **Given** the end pattern is running, **When** two or more players are alive, **Then** the round goes on, past the end of the configured time, until at most one is alive.
4. **Given** the end pattern, **When** it has covered the whole board, **Then** nobody can be alive on it, so the round has ended with the last survivor as the winner, or as a draw if the last players were eliminated by the same block or explosion.
5. **Given** a round where one player is left before the last 60 seconds, **When** they are the only one alive, **Then** the round ends at once as it does today and the pattern never starts.
6. **Given** the end pattern, **When** it is drawn, **Then** the page and the terminal show the marked squares, the landed blocks and the time left, and the page plays a warning and an impact sound (generated, no recordings).
7. **Given** the same settings, seed and inputs, **When** the round is played twice, **Then** the pattern, the timing and the result are identical.

---

### User Story 3 - Team mode, 2 against 2 (Priority: P3)

In Team mode the four participants form two teams of two (slots 1 and 2 against slots 3 and 4). The rules are those of Normal mode, including the end pattern, but a team wins when one of its members is alive and no member of the other team is. The lobby, the cards, the board and the status show who is on which team. With the friendly-fire switch on (the default) explosions hurt teammates as well; with it off, a team's own explosions never hurt its members.

**Why this priority**: It reuses the Normal rules and only adds the notion of teams and a team result, so it is the cheapest new mode.

**Independent Test**: Start a Team match with four bots, eliminate both members of one team, and check that the other team is announced as the winner even though both its members may be alive or only one; check that with friendly fire off a bomb placed by one member does not eliminate the other.

**Acceptance Scenarios**:

1. **Given** Team mode with four participants, **When** the lobby is shown, **Then** each card shows its team (name and colour marker) and the owner cannot start with fewer than four participants.
2. **Given** a Team match, **When** all members of one team are eliminated, **Then** the round ends and the other team is the winner, even if one of its members is also eliminated.
3. **Given** a Team match in which the last survivors of both teams are eliminated at the same time, **When** the round ends, **Then** it is a draw.
4. **Given** friendly fire off, **When** a member's explosion reaches a teammate, **Then** the teammate is not eliminated (and chain reactions still work); **Given** friendly fire on, **Then** the teammate is eliminated as any other player would be.
5. **Given** a Team match, **When** it is drawn or described, **Then** teammates are marked in the page and the terminal, the status lists the teams, and the result message names the winning team and its members.
6. **Given** a human leaves during a Team match, **When** a bot takes their slot, **Then** the bot plays for the same team and bots never target their teammates on purpose.

---

### User Story 4 - Chaser mode (Priority: P4)

In Chaser mode one participant, chosen at random for each round, is the chaser and the other three are runners. The arena is nearly open: far fewer walls and blocks than in Normal mode. The chaser starts in the center with 5 bombs and 3 hearts and has to eliminate the runners with explosions. The runners have no bombs and can only run. Bonuses appear at random squares of the map from time to time (blocks do not hide them). A runner is eliminated by one explosion; the chaser loses one heart per explosion that reaches them, even their own, and is protected for a short moment after losing a heart. The chaser wins when all runners are eliminated; the runners win if the time runs out while at least one of them is alive, or when the chaser has lost all hearts.

**Why this priority**: It is the biggest rules change (roles, hearts, no bombs for most players, random bonuses, another win condition), and the other stories do not depend on it.

**Independent Test**: Start a Chaser match with bots, check that exactly one participant starts in the center with 5 bombs and 3 hearts, that the others start with no bombs and cannot place any, that bonuses appear on their own, and that each of the three endings (chaser kills all, time runs out, hearts run out) gives the right result.

**Acceptance Scenarios**:

1. **Given** Chaser mode with four participants, **When** the round starts, **Then** one of them (chosen with the round seed) is the chaser, in the center of the board with 5 bombs and 3 hearts, and the other three are runners in the corners; the arena has clearly fewer obstacles than in Normal mode and every square is reachable.
2. **Given** a runner, **When** they press the bomb key, **Then** nothing happens; the extra-bomb bonus never appears in this mode, so runners never get bombs.
3. **Given** a Chaser round, **When** time passes, **Then** bonuses appear on random free squares at a regular pace, at most a few at a time, and disappear when picked up or destroyed.
4. **Given** the chaser, **When** an explosion (anybody's, including their own) reaches them, **Then** they lose one heart and are protected, shown by blinking, for about 2 seconds, during which further explosions do not cost hearts; **When** the last heart is lost, **Then** the chaser is eliminated and the runners win.
5. **Given** a runner, **When** an explosion reaches them, **Then** they are eliminated at once; **When** the last runner is eliminated, **Then** the chaser wins.
6. **Given** a Chaser round, **When** the configured time runs out and at least one runner is alive and the chaser has hearts left, **Then** the round ends and all living runners win; the end pattern does not run in this mode.
7. **Given** the runners pick up a kick, punch, roller or malus, **When** they use it, **Then** it works as in the other modes (so a runner with the punch or the kick can send the chaser's bombs back at them); the flame and extra-bomb bonuses do not appear in this mode.
8. **Given** the chaser's or a runner's slot is held by a bot, **When** the round runs, **Then** the chaser bot hunts runners and places bombs only when it has a way out, runner bots flee explosions and the chaser's bombs and collect bonuses, and none of them stand still or get stuck.
9. **Given** Chaser mode, **When** it is drawn and described, **Then** the chaser is marked (badge or name tag), hearts are shown as hearts in the page and as a count in the terminal, and the result message says "Chaser wins" or "Runners win".

---

### Edge Cases

- The owner picks Team or Chaser while fewer than four participants are in the lobby: the settings are accepted, the start is refused until four are present (bots can fill the slots), and the lobby says so.
- The owner changes the mode while participants are ready: the ready marks are kept; the lobby shows the changed mode to everybody.
- A participant leaves in the middle of a match: a bot takes the slot and keeps the team, the role, the hearts and the bombs of the leaver; the room stays open while at least one human is left.
- The only human leaves while bots play on: the match ends as it does today and the room closes; the next person to join creates a new room.
- The time setting is shorter than the end pattern needs to be useful (2 minutes): the pattern still starts at 60 seconds before the end; nothing starts earlier.
- Two blocks are due on one square or a block is due on a square that is already a block or a wall: it is skipped without delay, and the pattern order stays fixed.
- Blocks never land on the chaser: Chaser mode has no end pattern, so hearts only protect against explosions.
- A bomb is sliding or flying when its square is covered by the end pattern: it is destroyed with the square if it is on it at the moment of landing; a flying bomb is unaffected until it lands.
- Friendly fire off in a Team mode where a teammate's bomb also hurts an opponent standing next to the teammate: only the opponent is eliminated.
- A very small terminal with a large board: the terminal shows the existing "terminal too small" message with the size needed; the room settings can still be changed.
- Several sizes with the standard fairness rules: the start corners and their neighbouring squares are free on every size, every free square can be reached from every start, and blocks never fully enclose a player.
- Developer mode is on: pausing bots and the drop table still work in Normal and Team; the drop table has no effect in Chaser mode (bonuses appear on their own), and the developer panel says so.
- The room name is empty, too long or contains control characters: it is replaced by a default or refused with a clear message, and never breaks the lobby.

## Requirements *(mandatory)*

### Functional Requirements

**Room creation and settings**

- **FR-001**: Before the first person plays on a server, the page MUST show a room creation menu where they set the room name, the mode, the board size, the round time and (for Team mode) friendly fire, with the defaults Normal, small board, 3 minutes and friendly fire on.
- **FR-002**: The person who creates the room MUST become its owner; the owner of a room is the lowest slot held by a person, as today, and MUST be the only one who can change the settings or start the match.
- **FR-003**: Everybody in the room MUST see the room name, the mode and the settings in the lobby, in the page and in the terminal, and MUST see changes at once.
- **FR-004**: The owner MUST be able to change the settings in the lobby; changes by anyone else, and changes during a match, MUST be refused with a clear message and no effect.
- **FR-005**: The settings MUST be one self-contained description of the room (name, mode, size, time, friendly fire) that is kept for the rematch and can be handed to other parts of the system unchanged, so that a future feature can list and join rooms on the network; this feature MUST NOT add room lists, several rooms per server or passwords.
- **FR-006**: The board size MUST be one of three sizes: small (the current default and the smallest size the game allows), medium and large, each with odd dimensions, with the start corners, fairness and reachability rules of the existing arena on every size.
- **FR-007**: The round time MUST be one of 2, 3, 5 and 10 minutes, with 3 minutes as the default (the current round length).
- **FR-008**: Each mode MUST state how many participants it needs; Normal needs 1 to 4 as today, Team and Chaser need exactly 4; a start with fewer MUST be refused with a message, and bots MUST be able to fill the slots.
- **FR-009**: Unknown or invalid settings values (from an old page, a hand-made message or an option) MUST be replaced by the defaults or refused, and MUST never break the room or other participants.
- **FR-010**: The terminal host MUST offer the same settings by command-line options and on a room setup screen before the lobby, and terminal participants MUST see them in the lobby; a terminal player who joins an empty room MUST get the defaults.

**End pattern (Normal and Team)**

- **FR-011**: In Normal and Team mode, during the last 60 seconds of the round time, blocks MUST fall on the board in a fixed pattern that starts at the outer ring and spirals inwards, until at most one player (or one team) is left.
- **FR-012**: Each falling block MUST be announced on its square for about one second before it lands; a block landing on a player MUST eliminate them, MUST destroy a bomb or power-up on the square without an explosion, and the square MUST become a permanent block that explosions cannot destroy.
- **FR-013**: The pattern MUST cover every free square within about 60 seconds after it starts, whatever the board size, so that no round lasts longer than the configured time plus about 60 seconds plus the announcement; squares that are already walls or blocks MUST be skipped without a pause.
- **FR-014**: In Normal and Team mode a round MUST NOT end because the time ran out while two or more players (or both teams) are alive; it MUST end with a draw only when the last players are eliminated together.
- **FR-015**: The page and the terminal MUST show the announcement, the landed blocks and the time left, and the page MUST play a generated warning sound and a generated impact sound under the existing sound settings.

**Team mode**

- **FR-016**: Team mode MUST put slots 1 and 2 in one team and slots 3 and 4 in the other, MUST show the teams in the lobby, in the match and in the status, and MUST announce the winning team.
- **FR-017**: A team MUST win when at least one of its members is alive and no member of the other team is; if the last members of both teams are eliminated together the round MUST be a draw.
- **FR-018**: With friendly fire on, explosions MUST eliminate teammates like any player; with friendly fire off, a player's explosions MUST NOT eliminate their teammates (a player can still be eliminated by their own explosion), while still triggering chain reactions and still hurting opponents.
- **FR-019**: Bots MUST play for their team, MUST NOT target teammates on purpose, and MUST keep escaping every explosion, including their teammates'.

**Chaser mode**

- **FR-020**: Chaser mode MUST give exactly one participant the role of chaser, chosen for each round from the round seed, and the other three the role of runner.
- **FR-021**: The chaser MUST start in the center of the board with 5 bombs and 3 hearts; the runners MUST start in the corners with no bombs and MUST NOT be able to place bombs by any means in this mode.
- **FR-022**: The arena of Chaser mode MUST have clearly fewer walls and destructible blocks than the Normal arena (about a third of the fixed walls and a small share of blocks), MUST be fully reachable and MUST leave room to run.
- **FR-023**: In Chaser mode bonuses and maluses MUST appear at random free squares at a regular pace with a small cap on the number on the board; the extra-bomb and flame bonuses MUST NOT appear; all other powers MUST work as in the other modes.
- **FR-024**: A runner MUST be eliminated by one explosion; the chaser MUST lose one heart per explosion that reaches them (their own too) and MUST be protected for about 2 seconds after losing a heart; with no hearts left the chaser MUST be eliminated.
- **FR-025**: The chaser MUST win when all runners are eliminated; the runners MUST win when the chaser is eliminated, or when the round time runs out while at least one runner and the chaser are alive; the end pattern MUST NOT run in Chaser mode.
- **FR-026**: The page and the terminal MUST mark the chaser, show the chaser's hearts and the protection, and announce "Chaser wins" or "Runners win"; the page MUST play a generated sound when a heart is lost.
- **FR-027**: Bots MUST play both roles: a chaser bot hunts runners and bombs only when it has a way out; runner bots flee explosions and the chaser's bombs and collect bonuses, and none of them may crash or get stuck.

**General**

- **FR-028**: All new rules MUST be the same in the page and the terminal (dual-interface parity), MUST be deterministic for a given seed and settings, and MUST keep protocol version 1 compatible for clients that do not know them (new fields are optional; a client that does not know a mode still sees a normal lobby and observation).
- **FR-029**: The added messages and fields MUST be validated like all other external input (wrong types, out-of-range values, oversized strings) with no panics.
- **FR-030**: The README and the documentation MUST describe the room creation, every setting, the end pattern and each mode with its rules, controls and options, including the options of the terminal host.

### Key Entities *(include if feature involves data)*

- **Room settings**: Room name, mode, board size, round time and friendly fire; created in the room creation menu, changed by the owner in the lobby, shown to everybody and kept for rematches; the unit a future network feature will publish.
- **Mode**: Normal, Team or Chaser; defines the participants needed, the roles or teams, the arena, the way a round ends and which bonuses appear.
- **Team**: A group of two participants (slots 1 and 2, slots 3 and 4) that wins or loses together in Team mode.
- **Role**: Chaser or runner in Chaser mode; defines bombs, hearts and the win condition of a participant.
- **Hearts**: The chaser's remaining lives and the short protection after losing one.
- **End pattern**: The fixed order in which squares are announced and covered by blocks during the last 60 seconds of Normal and Team rounds, with its warnings and landings.
- **Random bonus schedule**: When and where bonuses appear in Chaser mode, with the cap on how many lie on the board.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: A person can open the room creation menu, set a mode, a size and a time and reach the lobby in under 30 seconds, and every participant sees the same settings in 100% of checks.
- **SC-002**: In 100% of attempts, a change of settings by a non-owner or during a match is refused with a message and leaves the settings unchanged.
- **SC-003**: With no change in the menu, a match plays exactly as before this feature (same arena for a seed, same rules) until the last 60 seconds, in 100% of checks.
- **SC-004**: In 100% of Normal and Team rounds played to the end with the end pattern, the round ends with at most one player or team alive within about 70 seconds after the pattern starts, on all three board sizes, and no player is alive on a covered square.
- **SC-005**: In 100% of 20 prepared end-pattern situations (player on a landing square, bomb, power-up, flame, wall already there, last two players eliminated together, the last player alone), the result is the expected one, and the same seed and inputs give the identical pattern and result.
- **SC-006**: In 100% of 20 prepared Team situations (each team eliminated in turn, both at once, teammate in a blast with friendly fire on and off, a chain reaction, a bot taking over), the result and the eliminations are the expected ones.
- **SC-007**: In 100% of Chaser rounds exactly one chaser starts in the center with 5 bombs and 3 hearts and no runner can ever place a bomb; in 20 prepared situations (each of the three endings, the protection after a hit, the last heart, own explosion, a runner punching back a bomb) the result is the expected one.
- **SC-008**: Over 200 simulated Chaser rounds with bots, every round ends before the time limit plus one second, no participant gets stuck for more than 5 seconds, and bonuses appear at the configured pace within 20%.
- **SC-009**: A person looking at a screenshot of a running match can tell the mode, the teams or the chaser and the hearts, and the announced squares of the end pattern, in 100% of cases.
- **SC-010**: With the three board sizes and all modes, key presses still show on screen within 200 ms at the 95th percentile and a match still runs smoothly on a typical laptop.
- **SC-011**: A new reader can find how to create a room, what each mode and setting does and the options of the terminal host in the documentation within 5 minutes.

## Assumptions

- There is one room per server in this feature; "room" is the existing lobby plus its settings. Several rooms, a room list, discovery on the network, passwords and moving players between rooms belong to a future specification; this one only keeps the settings in one self-contained description so that they can be published later.
- The room exists while at least one person is in it; when the last person leaves the lobby the room closes and the next first person creates a new one (with the defaults, or with the options the server or terminal host was started with).
- "Minimal size" is the current arena (15 x 13), which becomes the small size and the default; the other sizes are medium (19 x 15) and large (25 x 19). Sizes stay odd and within the limits the page and the protocol already accept.
- The round time is the current 3 minutes by default; Normal and Team mode need the extra time for the end pattern, Chaser mode uses it as the time the runners have to survive.
- Teams are fixed by slot (1 and 2 against 3 and 4) in this version; choosing teams is a possible later improvement. The room keeps four slots; Team mode is written with teams of two so that larger rooms can be added later.
- Friendly fire is a room setting (default on, the classic rule) rather than a fixed rule, which settles the question of whether teammates can hurt each other without taking the choice away from the owner.
- In Chaser mode the chaser is chosen at random with the round seed (so a rematch gets another chaser); the runners' bonuses are the speed, kick and punch bonuses and the three maluses, which lets a runner send bombs back at the chaser; extra bomb and flame are not offered so that runners stay without bombs.
- Chaser mode bonuses appear about every 5 seconds with at most 6 on the board; the chaser's protection after losing a heart lasts about 2 seconds; the arena has about a third of the fixed walls and a small share of destructible blocks. These numbers are tuning values that can change without changing the rules.
- The end pattern announces a square for about 1 second, starts with the outer ring and spirals inwards clockwise, and its pace is set so that the whole board is covered about 60 seconds after it starts, whatever the size.
- A block that lands on a bomb or a power-up destroys it without an explosion; a flying bomb is not affected until it lands.
- The existing powers, looks, sound settings and developer mode keep working; the drop table is used in Normal and Team mode only.
- Existing behaviour stays: up to 4 participants, host authority, the 20 Hz tick, bots, bomb fuse and chain reactions, and reproducibility for a given seed and settings.
- This feature builds on the web page, server, terminal client and protocol of the earlier features; added messages and fields are additive and protocol version 1 stays compatible.
