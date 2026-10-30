# Feature Specification: Bot Behaviours and Difficulty Levels

**Feature Branch**: `012-bot-behaviours-difficulty`

**Created**: 2026-10-30

**Status**: Draft

**Input**: User description: "Bots must have different behaviours when playing (agressive, or protective or mixing both) with different difficulties (configurable by owner of the room) (easy is a bit slower with mistakes, medium slow with few mistakes, then hard)". Refined at planning: "Add different behaviours for the bots, randomly chosen when starting a new game. Then implement difficulties (slower decisions taking, few mistakes). Then add a room setting so owner can choose bot difficulty (default is medium)."

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Bots play with distinct behaviours (Priority: P1)

A player starts a match with bots. Each bot is given one of three behaviours, picked at random when
the match starts: **Aggressive** (goes after opponents first, bombs them whenever it safely can),
**Protective** (keeps its distance from opponents and danger, collects power-ups and digs first,
attacks only opponents that come close) or **Balanced** (a mix of both, like today's bots). Two
matches with the same bots feel different because the bots do not all play the same way.

**Why this priority**: Varied opponents are the core value of the feature and work even before
difficulty levels exist.

**Independent Test**: Run many seeded matches and check that every behaviour is picked, and that in
scripted situations an Aggressive bot moves toward a reachable opponent while a Protective bot in
the same situation does not.

**Acceptance Scenarios**:

1. **Given** a match starts with bots, **When** the match begins, **Then** each bot is given one
   behaviour among Aggressive, Protective and Balanced, independently of the other bots.
2. **Given** the same match seed, **When** the match is replayed, **Then** every bot gets the same
   behaviour and makes the same decisions.
3. **Given** a bot with a free path to an opponent and nothing else worth doing nearby, **When** the
   bot is Aggressive, **Then** it heads toward a square from which its bomb would reach that
   opponent.
4. **Given** an opponent a few squares away and a power-up or block to dig nearby, **When** the bot
   is Protective, **Then** it goes for the power-up or block instead of the opponent and does not
   end its move next to the opponent.
5. **Given** a new match starts in the same room (rematch), **When** the bots are created, **Then**
   their behaviours are picked again.
6. **Given** a human leaves during a match and a bot takes over, **When** the bot starts playing,
   **Then** it also gets a randomly picked behaviour.
7. **Given** a room in developer mode, **When** the owner chooses Aggressive, Protective or Balanced
   as the bot behaviour in the developer panel, **Then** every bot created afterwards (next match,
   takeovers) uses that behaviour; choosing Random restores the random pick.
8. **Given** a room that is not in developer mode, or a member who is not the owner, **When** a
   bot behaviour is requested, **Then** the request is refused and bots keep the random pick.

---

### User Story 2 - Bots play at a chosen difficulty (Priority: P2)

Bots play at one of three difficulty levels. **Easy** bots react noticeably slower and make
mistakes regularly (wrong turns, missed or badly timed bombs). **Medium** bots react a bit slower
than the best bots and make a few mistakes. **Hard** bots react every moment and never make
deliberate mistakes. Difficulty is independent from behaviour: any behaviour can be played at any
difficulty.

**Why this priority**: Difficulty lets beginners enjoy the game against bots and keeps experienced
players challenged; it depends on nothing but the bots themselves.

**Independent Test**: Run many seeded bot-versus-bot matches and compare results by difficulty;
in scripted situations check that a slower bot waits before reacting and that a Hard bot reacts
immediately.

**Acceptance Scenarios**:

1. **Given** a bomb is placed near a Hard bot, **When** the next moment of play comes, **Then** the
   bot already reacts to it.
2. **Given** a bomb is placed near an Easy or Medium bot, **When** less than the bot's reaction
   time has passed, **Then** the bot has not reacted to it yet; once the reaction time has passed
   it reacts, and the Easy reaction time is longer than the Medium one.
3. **Given** many matches of a Hard bot against an Easy bot, **When** results are counted, **Then**
   the Hard bot wins clearly more often, and likewise Medium against Easy and Hard against Medium.
4. **Given** a Hard bot, **When** it plays, **Then** it never makes a deliberate mistake.
5. **Given** an Easy or Medium bot, **When** it makes a mistake, **Then** it never walks into a
   square that is burning at that moment.

---

### User Story 3 - The room owner chooses the bot difficulty (Priority: P3)

When creating or configuring a room, the owner (the host) picks the bot difficulty (Easy, Medium
or Hard, Medium by default). Nobody else can change it. Every member sees the chosen difficulty in
the lobby. All bots of the next match play at that difficulty, in both the terminal and the web
versions.

**Why this priority**: Gives control to the owner, but needs User Story 2 to have any effect.

**Independent Test**: Create a room, change the difficulty as owner, start a match and check that
every bot plays at that difficulty; check that a non-owner cannot change it.

**Acceptance Scenarios**:

1. **Given** a new room, **When** nobody changed the bot difficulty, **Then** it is Medium.
2. **Given** the owner in the lobby, **When** they choose Easy, Medium or Hard, **Then** every member
   sees the new value and the next match's bots use it.
3. **Given** a member who is not the owner, **When** they try to change the bot difficulty, **Then**
   the change is refused and the setting stays as it was.
4. **Given** a running match, **When** anybody tries to change the bot difficulty, **Then** the change
   is refused until the match ends.
5. **Given** a bot takes over a human's seat during a match, **When** it starts playing, **Then** it
   uses the room's bot difficulty.
6. **Given** a client or saved settings that do not mention the bot difficulty, **When** a room is
   created from them, **Then** the room uses Medium.
7. **Given** a terminal host, **When** the host sets the bot difficulty on the command line, in the
   room setup screen or in the lobby, **Then** the room's bots use it.

---

### Edge Cases

- A room with no bots still keeps and shows its bot difficulty; it applies once bots join.
- An unknown or malformed difficulty value from a client is ignored and Medium is used; it never
  closes the room or the connection.
- In Chaser mode, bot runners have no bombs: their behaviour only changes how they keep away from
  the chaser; a bot chaser's behaviour changes how eagerly it hunts.
- In Team mode, an Aggressive bot still never places a bomb that would reach a teammate while
  friendly fire is on.
- During the closing end pattern, every bot, whatever its behaviour and difficulty, still moves
  toward squares that close later.
- A slow (Easy) bot whose reaction time has not passed while a flame is already burning next to it
  still does not step into that flame.
- When a bot is trapped with no safe square, it behaves as today (steps toward the square that burns
  last), whatever its difficulty.
- The bot difficulty changes only between matches; changing it during a match has no effect on
  running bots.

## Requirements *(mandatory)*

### Functional Requirements

**Behaviours**

- **FR-001**: The system MUST provide three bot behaviours: Aggressive, Protective and Balanced.
- **FR-002**: When a match starts, each bot MUST be given one behaviour picked at random, each
  behaviour being equally likely, independently for each bot.
- **FR-003**: The behaviour pick MUST be reproducible: the same match seed and seats MUST give the
  same behaviours.
- **FR-004**: A bot that takes over a human's seat during a match MUST get a randomly picked
  behaviour as well.
- **FR-005**: An Aggressive bot MUST prefer reaching a square from which its bomb would hit an
  opponent over collecting power-ups or digging blocks, and MUST look for opponents farther away than
  a Balanced bot.
- **FR-006**: A Protective bot MUST prefer collecting power-ups and digging blocks over hunting, MUST
  only hunt opponents that are close, and MUST avoid ending a move adjacent to an opponent when
  another safe move exists.
- **FR-007**: A Balanced bot MUST keep the priorities of today's bots (power-ups, then opponents, then
  blocks).
- **FR-008**: Every behaviour MUST keep today's safety rules: fleeing danger first, never placing a
  bomb without a way out, respecting friendly fire in Team mode, and moving inward during the
  closing end pattern.
- **FR-009**: Behaviours MUST NOT give bots any information humans do not have.
- **FR-023**: In developer mode only, the room owner MUST be able to force the behaviour of the
  room's bots to Aggressive, Protective or Balanced, or set it back to Random (the default).
- **FR-024**: A forced behaviour MUST apply to every bot created after the change (next match
  start and mid-match takeovers) and MUST be shown to every member in the developer panel.
- **FR-025**: Behaviour requests from non-owners, from rooms not in developer mode, or with an
  unknown behaviour MUST be refused with an error and change nothing.

**Difficulty**

- **FR-010**: The system MUST provide three bot difficulty levels: Easy, Medium and Hard.
- **FR-011**: A bot's reaction time to new dangers and opportunities MUST be longest at Easy, shorter
  at Medium and zero at Hard (Hard reacts on every game tick, like today's bots).
- **FR-012**: Easy and Medium bots MUST make occasional deliberate mistakes (such as a wrong turn, a
  missed bombing opportunity or a late reaction), more often at Easy than at Medium; Hard bots MUST
  NOT make deliberate mistakes.
- **FR-013**: Bots at every difficulty MUST NOT step onto a square that is burning at that moment,
  even when making a mistake.
- **FR-014**: Mistakes and reaction delays MUST be reproducible from the match seed.
- **FR-015**: A Balanced bot at Hard MUST play exactly like today's bot.

**Room setting**

- **FR-016**: Rooms MUST have a bot difficulty setting with the values Easy, Medium and Hard; its
  default MUST be Medium.
- **FR-017**: Only the room owner MUST be able to change the bot difficulty, and only before a match
  starts; the server MUST enforce this whatever the client shows.
- **FR-018**: Every room member MUST see the current bot difficulty in the lobby, in both the web
  and terminal clients.
- **FR-019**: All bots of a match, including those that take over for a human mid-match, MUST play at
  the room's bot difficulty as it was when the match started.
- **FR-020**: Clients and saved settings that do not send a bot difficulty MUST keep working and get
  Medium; invalid values MUST be ignored without error.
- **FR-021**: Only the host (room owner) MUST be able to choose the bot difficulty: the terminal host
  through its command line, room setup screen and lobby, and a web owner through the room creation
  form and the lobby configuration panel. Terminal local games and the headless web server offer
  no difficulty control.
- **FR-022**: The bot difficulty and the bot behaviours MUST be documented for players.

### Key Entities

- **Bot behaviour**: The play style of one bot for one match (Aggressive, Protective, Balanced);
  picked at random at match start; not a room setting and not chosen by players, except that the
  owner of a developer-mode room may force it for testing.
- **Bot difficulty**: A room setting (Easy, Medium, Hard; default Medium) shared by every bot of the
  room's matches; defines reaction time and mistake frequency.
- **Room settings**: Existing owner-chosen values of a room (name, mode, board size, round time,
  friendly fire), extended with the bot difficulty.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Over 300 seeded match starts with bots, each behaviour is picked for between 25% and
  42% of bots.
- **SC-002**: In 100 seeded one-against-one bot matches with the same behaviour, a Hard bot wins at
  least 65% of decided rounds against an Easy bot; Medium wins at least 55% against Easy and Hard
  wins at least 55% against Medium.
- **SC-003**: Easy bots eliminate themselves at least twice as often as Hard bots over the same 100
  seeded four-bot matches.
- **SC-004**: In 100 seeded four-bot matches, Aggressive bots place bombs that reach an opponent at
  least 30% more often than Protective bots, and Protective bots survive longer on average than
  Aggressive bots.
- **SC-005**: A room owner can change the bot difficulty in under 5 seconds from the lobby, and
  100% of attempts by non-owners or during a match are refused.
- **SC-006**: Existing clients that do not know the bot difficulty can still create, join and play
  rooms with no visible error.

## Assumptions

- Behaviours are not shown to players during the match; they are discovered by playing. Showing
  them is out of scope.
- Behaviours are picked uniformly at random; the owner does not choose or restrict them outside
  developer mode. Developer mode exists only in the web page (as today), so the behaviour override
  has no terminal control.
- Terminal local games keep the default Medium difficulty.
- One difficulty applies to all bots of a room; per-bot difficulty is out of scope.
- Reaction times stay short enough (well under the bomb fuse) that Easy bots still usually escape
  their own bombs; Easy is meant to be beatable, not suicidal.
- Today's bot becomes the Balanced behaviour at Hard difficulty, so the existing bot tests keep
  their meaning.
- The legacy single-room host and the multi-room server both honour the setting.
- No new persistence: the web client already remembers room settings locally and simply includes the
  new value.
