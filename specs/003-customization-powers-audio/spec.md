# Feature Specification: Player Customization, New Powers, Sound and Developer Mode

**Feature Branch**: `003-customization-powers-audio`

**Created**: 2026-10-22

**Status**: Draft

**Input**: User description: "I want users to be able to customize the player they have with colors, gender, some accessories. Improve lobby view also with isometric design instead of plain 2d. See image for reference. Add some noise and music (generated ideally. No copyright infragments). Also add several Powers: + roller icon = more speed + another icon to increase flames size + A punch to be able to punch bombs (pretty rare) - implement this feature. In this case the bomb will jump square per square until a free square (no block or player). Other powers to implement: + Slow to slow a bit the player + Hyper Speed + Small bomb to get a 1 square bomb. Add Settings for sound level. Add also a dev mode to be able to pause bots and configure dropping for each bonus/malus"

**Relationship to 001 and 002**: This feature extends the game of `001-bomberman-multiplayer` (rules, bots, network play) and the web page of `002-web-client`. The two existing power-ups (more speed, longer flames) and the extra bomb stay and get proper icons; new powers are added to the same rules for every client. The lobby cards of the web page are redesigned after the reference image (character cards with a three-dimensional, cubic look). The in-match board keeps its current top-down look.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Make my player look like me and see everybody in a better lobby (Priority: P1)

Before a match, a person opens a character editor from the join screen or the lobby. They pick a main colour, a gender look, and accessories (for example a hat and glasses), and see the character update immediately. When they join, the lobby shows every participant on a character card in an isometric, cubic style (as in the reference image: tall card, large character, name banner, player name below). In the match, each player is drawn with the look they chose.

**Why this priority**: It is the most visible change and the reason people will want to come back; it needs no new game rules.

**Independent Test**: Open the page, change colour, gender and accessories, join a server with a bot, check the lobby card and then the match for the same look, then open a second browser and check that it sees the first player's look.

**Acceptance Scenarios**:

1. **Given** the join screen, **When** the user changes colour, gender or an accessory, **Then** a preview of the character shows the change at once.
2. **Given** a chosen look, **When** the user reloads the page later, **Then** the same look is selected again without choosing it twice.
3. **Given** a lobby with several participants, **When** it is shown, **Then** each occupied slot is a character card in the isometric, cubic style with that participant's look, a name banner and the player name below, and empty slots are shown as empty cards.
4. **Given** two participants choose the same main colour, **When** both are in the lobby, **Then** the later one is given the nearest free colour and told so, so that colours stay distinct.
5. **Given** a started match, **When** it is drawn, **Then** every player appears with their chosen colour, gender look and accessories, and the four players can still be told apart at a glance.
6. **Given** a joined player in the lobby, **When** they change their look before the match starts, **Then** everybody sees the new look; **When** the match has started, **Then** the editor is locked until the lobby returns.
7. **Given** a terminal player or a bot, **When** they take a slot, **Then** they get a sensible default look so every card and every player is drawn.
8. **Given** a running match, **When** a player moves up, down, left or right, **Then** their sprite shows the matching view (front when moving down or standing, back when moving up, side when moving left or right, mirrored for the other side), with the same colour, gender look and accessories seen from that side (for example no face from behind, a hat brim and one lens from the side).
9. **Given** a player who is moving, **When** the match is drawn, **Then** the sprite plays a walking animation (alternating feet and arm swing, a small bob) at a pace that follows the movement, and **When** the player stands still, **Then** it shows an idle pose in the last direction faced.

---

### User Story 2 - Play with a richer set of powers, including the punch (Priority: P2)

Destroyed blocks may reveal powers on the board. Besides the existing extra bomb, the match now has: a roller (more speed), a flame (longer explosions), a punch (rare; lets the player punch bombs away), a kick (a bonus that lets the player push bombs by walking into them), and three maluses: slow (the player moves slower for a while), hyper speed (the player moves far too fast for a while) and small bomb (bombs only reach 1 square for a while). Every power has its own pixel-art icon (a small picture in the style of the rest of the board, not a letter), and bonuses and maluses are easy to tell apart. A punched bomb hops one square at a time in the direction of the punch over anything in the way, and lands on the first free square (no block, wall, player or other bomb). A kicked bomb slides in the direction of the push until the next square is blocked.

**Why this priority**: It changes how matches are played and is a clear rules change, but it is independent of looks and sound.

**Independent Test**: With a fixed seed and a prepared arena, pick up each power in turn and check its effect on speed, range, bomb reach and punching; punch a bomb across blocks and players and check where it lands.

**Acceptance Scenarios**:

1. **Given** a power on the board, **When** a player steps on it, **Then** it disappears, its effect applies at once, and the player's status line shows it.
2. **Given** a roller or a flame pickup, **When** the player keeps moving or bombing, **Then** they move faster or their explosions are one square longer, up to a limit, as for the existing speed and range powers.
3. **Given** a player with the punch, **When** they press the punch key facing a bomb on the next square, **Then** the bomb leaves its square and hops square by square in that direction, passing over blocks, walls, players and other bombs, and stops on the first free square; its fuse keeps burning during and after the flight.
4. **Given** a punched bomb with no free square before the edge of the arena, **When** it would run out of squares, **Then** it stays on its original square and nothing is lost.
5. **Given** a player without the punch, **When** they press the punch key, **Then** nothing happens.
6. **Given** a slow pickup, **When** it is taken, **Then** the player moves noticeably slower for a limited time and then returns to their normal speed.
7. **Given** a hyper speed pickup, **When** it is taken, **Then** the player moves much faster than normal for a limited time and then returns to their normal speed.
8. **Given** a small bomb pickup, **When** it is taken, **Then** every bomb the player drops for a limited time reaches exactly 1 square in each direction, and afterwards their normal range returns.
9. **Given** a power that appears in a match, **When** it is on the board, **Then** it is drawn with its own pixel-art icon, bonuses and maluses are marked differently, and it works the same for browser, terminal and bot players.
10. **Given** a player with the kick, **When** they walk into a landed bomb, **Then** the bomb slides away from them one square at a time in that direction and stops on the last square before the next blocked square (block, wall, another bomb, a living player or the edge of the arena); its fuse keeps burning and it explodes where it is when the fuse ends.
11. **Given** a player without the kick, **When** they walk into a bomb, **Then** nothing happens (the bomb blocks them as before).
12. **Given** a bomb that is sliding, **When** the match is drawn or a bot looks at the board, **Then** the slide is visible as movement, and bots treat both the bomb's current square and the square where it will stop as dangerous.

---

### User Story 3 - Hear the game and set the volume (Priority: P3)

The page plays short sound effects for the things that happen in a match, and a looping background tune in the lobby and another during a match. All sounds and music are created by the game itself (generated), so there are no third-party recordings and no copyright issues. A settings screen lets each person set the volume of music and effects and mute everything; the choice is remembered.

**Why this priority**: It adds atmosphere and polish but does not change the rules.

**Independent Test**: Open the page, press a key to allow sound, play a short match with a bot and listen for the effects and both tunes, then change the volumes and mute and check the effect immediately and after a reload.

**Acceptance Scenarios**:

1. **Given** the page and the first key press or click, **When** the lobby is shown, **Then** the lobby tune starts and loops; **When** a match starts, **Then** the match tune replaces it, and the lobby tune returns with the lobby.
2. **Given** a running match, **When** a bomb is placed, a bomb explodes, a power is picked up (a different sound for a bonus and a malus), a bomb is punched or lands, a player is eliminated, and the round ends, **Then** a distinct sound effect plays for each.
3. **Given** the settings screen, **When** the user moves the music or effects slider, **Then** the loudness changes at once for that kind of sound only; **When** they mute, **Then** nothing is heard.
4. **Given** chosen volume settings, **When** the page is reloaded, **Then** the same settings apply.
5. **Given** a browser or device without sound, or sound blocked, **When** the game is played, **Then** it works completely without sound and without errors.
6. **Given** the page is in a background tab, **When** it is hidden, **Then** the music pauses and resumes when the tab is visible again.

---

### User Story 4 - Developer mode to test powers and bots (Priority: P4)

A developer starts the server in developer mode. In the web page, a clearly marked developer panel lets them pause and resume all bots, and set how often each of the seven powers (bonuses and maluses) drops from destroyed blocks, so that a given power can be made to appear all the time, never, or somewhere in between. A server that was not started in developer mode refuses all of this.

**Why this priority**: It is a tool for testing and balancing and is only useful once the powers exist.

**Independent Test**: Start the server in developer mode with bots, pause the bots and watch them stop, set one power to always drop and destroy blocks to see only that power, then start a normal server and check that the panel is not offered and the commands are refused.

**Acceptance Scenarios**:

1. **Given** a server started in developer mode, **When** the page is opened, **Then** a developer panel and a visible "dev mode" marker are available to the lobby owner.
2. **Given** a match with bots, **When** the developer pauses bots, **Then** every bot stops moving and dropping bombs within one second, bombs already placed keep burning, and human players are unaffected; **When** bots are resumed, **Then** they act again.
3. **Given** the developer panel, **When** a power's drop chance is set to 100% and all others to 0%, **Then** every destroyed block that drops something drops that power; **When** it is set to 0%, **Then** that power never appears.
4. **Given** changed drop chances, **When** blocks are destroyed afterwards, **Then** the new chances apply, even in the middle of a match, and a "reset to defaults" action restores the normal values.
5. **Given** a server that is not in developer mode, **When** a client sends a developer command, **Then** it is refused with a clear message and nothing changes, and the page shows no developer panel.
6. **Given** a developer-mode server, **When** it starts, **Then** it prints a warning that developer mode is for testing and should not be exposed to untrusted networks.

---

### Edge Cases

- A kicked bomb has no free square behind it (block, wall, bomb, player or edge right behind it): it does not move and nothing is lost.
- A player steps in front of a sliding bomb: the bomb stops on the square before the player; a player standing in the path stops it the same way.
- A sliding bomb is hit by flames: it explodes where it is, with the usual chain reactions; a sliding bomb that reaches the end of its fuse explodes where it is.
- A player with the kick pushes a bomb that is flying after a punch: nothing happens, only landed or sliding bombs can be kicked; punching a sliding bomb that is next to the puncher turns it into a flight.
- A punched bomb passes over a square that has a player, a block, a wall, another bomb or a flame; it lands only on a square with no block, wall, player or bomb (a burning square is allowed, the bomb will burn on its fuse).
- A player walks onto the square where a punched bomb is about to land; the bomb lands on the next free square instead.
- A punched bomb is flying when its fuse runs out; it explodes where it is at that moment, or at its landing square if it has landed, and chain reactions work as usual.
- A player is eliminated while holding the punch or a malus; their effects end with them and nothing carries over to the next round.
- A player picks up a malus while another is active; the new malus replaces the old one and its timer starts over. Picking up the same malus again restarts the timer.
- A power is destroyed by an explosion while lying on the board; it disappears as in the existing rules.
- Hyper speed or slow is active when the round ends; the lobby and the next round start with normal speed.
- Two players choose the same colour at the same moment; the server decides in a fixed order and tells the later one.
- An invalid or unknown look (for example from an old page or a hand-made message) is replaced by a default look and never breaks the lobby.
- A very long or odd player name is shown without breaking the card layout.
- The page is opened in a browser that blocks sound until the first interaction; no error is shown and the music starts at the first key press or click.
- The developer sets drop chances that add up to more than 100%; they are scaled down proportionally so the total never exceeds 100%.
- All four bots are paused and no human acts; the match continues until the time limit as usual.
- A terminal player joins a match with powers they cannot see as icons; they still see the power by a letter and a colour, and the status line lists their active effects.

## Requirements *(mandatory)*

### Functional Requirements

**Customization and lobby**

- **FR-001**: Users MUST be able to choose, for their own player, a main colour, a gender look (at least a male and a female look) and accessories (at least two accessory places, such as head and face, each with at least four choices plus "none").
- **FR-002**: The character editor MUST show a live preview that changes at once when any choice changes.
- **FR-003**: The chosen look MUST be remembered on the user's device and selected again on the next visit.
- **FR-004**: The chosen look MUST be sent with the join and shown to every participant, in the lobby and in the match, for browser, terminal and bot participants alike.
- **FR-005**: The four players of a match MUST always have distinct main colours; when a chosen colour is taken, the later participant MUST receive the nearest free colour and be told.
- **FR-006**: A player MUST be able to change their look while in the lobby and see everybody updated; changes MUST be refused during a match.
- **FR-007**: Unknown or invalid look values MUST be replaced by defaults and never cause an error for other participants.
- **FR-008**: Terminal players and bots MUST get a default look that follows their slot, so every slot can be drawn.
- **FR-009**: The lobby MUST show each slot as a tall character card in an isometric, cubic style after the reference image: a large character, a name banner with the slot or character name, and the player name under the card; empty slots MUST appear as empty cards; the owner, the local user and ready state MUST be marked.
- **FR-010**: In a match, each player MUST be drawn with the colour, gender look and accessories they chose, and the four players MUST remain easy to tell apart at normal game size.
- **FR-034**: Each player sprite in the match MUST have a front, a back and a side view (the other side mirrored) that keep the player's colour, gender look and accessories, and MUST show the view that matches the direction the player moves or last moved; at the start of a round the sprite faces down.
- **FR-035**: A moving player MUST be drawn with a walking animation (at least two alternating step frames, arm swing and a small bob) whose pace follows the player's speed, and a standing player MUST show an idle frame; the animation MUST need no extra data on the wire beyond the position and facing already shared.

**Powers**

- **FR-011**: The game MUST have these powers: extra bomb (existing), roller (more speed), flame (longer explosions), punch (rare), kick (push bombs), slow, hyper speed and small bomb. Each MUST have its own pixel-art icon on the board (a drawn picture, not a letter, falling back to a letter only when drawn too small to read); bonuses (extra bomb, roller, flame, punch, kick) and maluses (slow, hyper speed, small bomb) MUST be visibly distinguished.
- **FR-012**: A player picking up a power MUST get its effect at once, and the power MUST leave the board; the player's status MUST list their active powers and maluses.
- **FR-013**: Roller and flame MUST add one step of speed and one square of explosion length per pickup, up to the same limits as the current speed and range powers.
- **FR-014**: A player holding the punch MUST be able to use a punch action (its own key) on the bomb on the square they face; a player without the punch MUST not be able to; the punch MUST be kept until the end of the round and be usable any number of times.
- **FR-015**: A punched bomb MUST leave its square and move one square at a time in the direction of the punch, passing over blocks, walls, players and other bombs, and MUST land on the first square that has no block, wall, player or bomb; if there is none before the edge of the arena, the bomb MUST stay where it was. The flight MUST be shown as visible hops and the fuse MUST keep burning during it.
- **FR-016**: Slow MUST reduce the player's speed by about a quarter, hyper speed MUST raise it to about double the normal top speed, and small bomb MUST make every bomb the player drops have a reach of exactly 1 square; each malus MUST last 10 seconds, be replaced by a newer malus, restart when picked up again, and end at elimination or at the end of the round.
- **FR-017**: Powers MUST work the same for browser, terminal and bot players, and the terminal MUST show them by letter and colour and list the active effects in its status line.
- **FR-018**: Bots MUST handle the new rules: they MUST pick up bonuses, and MAY avoid maluses; they MUST not get stuck or crash on them.
- **FR-019**: For a given seed and the same inputs, the same powers MUST appear in the same places (the existing reproducibility MUST be kept).
- **FR-032**: A player holding the kick MUST push a landed bomb by walking into it: the bomb MUST slide one square per fixed short interval in the direction of the walk and stop on the last square before the next blocked square (block, wall, bomb, living player or the edge of the arena); if the square behind the bomb is already blocked it MUST not move. A sliding bomb MUST block movement, keep its fuse burning, explode with flames as usual and explode where it is. The kick MUST be kept until the end of the round, MUST work the same for browser, terminal and bot players (bots do not use it on purpose) and MUST be listed in the status line.
- **FR-033**: Bots MUST treat a sliding bomb as dangerous at its current square and at the square where it will stop.

**Drop rates**

- **FR-020**: Each destroyed block MUST drop at most one power; the chance that it drops a given power MUST be set per power, from 0% to 100%, and the defaults MUST make the punch rare (at most 5% of all drops) and each of the maluses less frequent than each of the common bonuses.

**Sound and music**

- **FR-021**: The page MUST play distinct short sound effects for: placing a bomb, an explosion, picking up a bonus, picking up a malus, punching or kicking a bomb, a punched bomb landing, a player being eliminated, the round result, and pressing buttons in the lobby.
- **FR-022**: The page MUST play a looping lobby tune in the lobby and a different looping tune during a match.
- **FR-023**: All sound effects and music MUST be original or generated by the game itself, with no third-party recordings or tunes, and the origin of every sound MUST be documented.
- **FR-024**: A settings screen MUST offer separate volume controls for music and for effects, and a mute for everything; changes MUST apply at once and be remembered on the device.
- **FR-025**: Sound MUST start only after the user's first interaction, MUST pause in a hidden page, and the game MUST be fully usable if sound is blocked or not available.

**Developer mode**

- **FR-026**: The server MUST offer a developer mode that is off by default and switched on only by an explicit start-up option; the page MUST show a developer panel and a visible marker only when it is on.
- **FR-027**: In developer mode, the lobby owner MUST be able to pause and resume all bots; paused bots MUST stop moving and placing bombs within one second while bombs already on the board keep burning, and human players MUST be unaffected.
- **FR-028**: In developer mode, the lobby owner MUST be able to set the drop chance of each of the eight powers from 0% to 100%, apply it to blocks destroyed from then on (also during a match), and restore the defaults with one action; if the chances add up to more than 100%, they MUST be scaled down proportionally.
- **FR-029**: A server not in developer mode MUST refuse every developer command with a clear message and no change; developer mode MUST print a warning at start-up that it is for testing and not for untrusted networks.

**General**

- **FR-030**: The added messages and fields MUST be validated like all other external input: wrong types, out-of-range values and oversized data MUST be refused or replaced with defaults without affecting other players.
- **FR-031**: The README and documentation MUST describe customization, the powers and the punch rules, the settings, how sounds are generated and where they come from, and developer mode with its options.

### Key Entities

- **Look**: A player's chosen appearance: main colour, gender look, and accessories per accessory place. Attached to a participant, shared with everybody, defaulted when missing or invalid.
- **Power**: A pick-up on the board that is either a bonus or a malus. Kinds: extra bomb, roller, flame, punch, kick, slow, hyper speed, small bomb. Has an icon, a drop chance and an effect.
- **Active effect**: A power that currently applies to a player: permanent for the round (bonuses) or timed (maluses).
- **Punched bomb**: A bomb in flight after a punch: direction, hops done, landing square, fuse remaining.
- **Sliding bomb**: A landed bomb that was kicked: direction, ticks until the next step, fuse remaining; it ends on the last free square before a blocked one.
- **Drop table**: The chance per power that a destroyed block drops it; defaults, and developer overrides on a developer-mode server.
- **Audio settings**: Music volume, effects volume and mute, kept per device.
- **Developer state**: Whether developer mode is on, whether bots are paused, and the current drop table.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: A user can pick colour, gender look and two accessories and join in under 1 minute, and sees every change in the preview within 1 second.
- **SC-002**: In a match of 4 participants, 100% of looks seen by every participant are identical to what each player chose (after any colour change by the server), in the lobby and in the match.
- **SC-003**: In a picture of a running match, a viewer can name the owner of each of the four characters by colour alone in 100% of cases, and each of the eight powers is recognisable by its pixel-art icon.
- **SC-004**: In 20 prepared punch situations (free next square, blocks, players, bombs, edge of arena, flames), 100% end with the bomb on the expected square, and the bomb never lands on a block, wall, player or another bomb.
- **SC-005**: Over 1000 simulated block destructions with default settings, the share of each power is within 3 percentage points of its configured value and the punch is at most 5% of drops.
- **SC-006**: With the drop chance of one power set to 100%, every destroyed block that drops something drops that power; set to 0%, it never appears, in 100% of 200 simulated destructions.
- **SC-007**: A timed malus ends within 0.5 seconds of its 10-second limit in 100% of checks, and the player's speed or bomb reach is back to normal afterwards.
- **SC-008**: Changing a volume control changes the loudness within 0.2 seconds, mute silences all sound, and the settings are the same after a reload in 100% of checks.
- **SC-009**: 100% of sounds and tunes shipped with the game have a documented generated or original source; no third-party recording is included.
- **SC-010**: With all of this switched on, a match still runs smoothly (no visible stutter at 60 frames per second on a typical laptop) and key presses show on screen within 200 ms at the 95th percentile.
- **SC-011**: In developer mode, bots stop within 1 second of pausing in 100% of checks; on a normal server, 100% of developer commands are refused.
- **SC-012**: A new reader can find how looks, powers, sound settings and developer mode work in the documentation within 5 minutes.
- **SC-013**: In 20 prepared kick situations (free run to a wall, a block, another bomb, a player, the edge, no room behind the bomb, a player without the kick), 100% end with the bomb on the expected square and a sliding bomb never enters a blocked square.
- **SC-014**: In a picture of a match, a viewer can tell for every moving player which way they are going (front, back, left or right view) in 100% of cases, and two pictures a fraction of a second apart show different step frames for a moving player and the same pose for a standing one.

## Assumptions

- Gender in the editor is a cosmetic choice of character shape and details (for example hair and face); it has no effect on rules.
- The reference image is read as the lobby style: tall cards with a large cubic, isometric character, a coloured name banner and the player name below. The in-match board keeps the current top-down view; only characters, the lobby and the icons change.
- Looks are stored on the user's device (no accounts) and are sent with the join; a returning user gets their last look again.
- The look is a web-page feature; terminal players are shown with a default look by slot and see powers by letters and colours; sound and music are web-page only.
- "Gender" offers a male and a female look, plus the default; additional options may be added later.
- Colours come from a fixed palette of at least 8 colours; the four players of a match always have different main colours.
- Bonuses (extra bomb, roller, flame, punch, kick) last for the whole round; maluses (slow, hyper speed, small bomb) are timed at 10 seconds, are not stackable, and the newest replaces the older one.
- The punch has no limit of uses and has no cooldown beyond the time a bomb needs to land; its key is a new key next to the bomb key (for example a letter key) and is shown in the controls help.
- Default drop shares among the powers that drop (adding up to the existing overall drop rate of blocks): roller 22%, flame 22%, extra bomb 19%, small bomb 11%, slow 10%, hyper speed 8%, kick 5%, punch 3%.
- The kick is the classic "push the bomb" bonus. It was requested under the name "roller", which this spec already uses for the speed bonus (rollerskates, as originally asked), so the new bonus is called **kick** in the game, the code and the docs; only the label needs to change if another name is preferred. It is picked up like any bonus, kept for the round, needs no key (walking into the bomb is the action) and slides a bomb one square every 3 ticks (0.15 s) until the next square is blocked.
- Player sprites keep the existing flat top-down pixel-art style; the side view is drawn facing left and mirrored for right, and the walk cycle has two step frames plus the idle frame. The facing and the position (already in the shared state) are enough to choose the frame.
- Power icons are small pixel-art pictures (an extra bomb icon, rollerskate, flame, boxing glove, boot, snail, lightning, a tiny bomb) drawn by the render module; the terminal keeps letters and colours.
- In developer mode the pause affects bots only; match time keeps running.
- Developer mode is for local testing; the developer panel is for the lobby owner only and is available in the web page, not in the terminal client.
- Sounds and music are generated by the game at run time from simple building blocks (tones, noise and rhythm patterns); no sound files from other people are used. Moods: a light, bouncy tune for the lobby and a faster tune for the match.
- Existing behaviour stays: 4 participants, host authority, the 20 Hz tick, bots, bomb fuse and chain reactions, reproducibility for a given seed.
- This feature depends on the web page, server and protocol of `002-web-client`; the added messages are additive and the protocol version stays compatible for clients that do not know them.
