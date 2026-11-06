# Feature Specification: Revenge Karts

**Feature Branch**: `main` (existing branch; no branch created)

**Created**: 2026-11-02

**Status**: Draft

**Input**: User description: "In team mode and normal mode (not chaser), when a player is killed (before and until the walls are closing). They re-appears in a small kart around the board like old bomberman. They can only move left/right to control the kart that will turn around the board and throw bombs to the second square in front of them (if there is a player or wall or bomb it will go to the next free available square, like a punch). Throwing on a player immobilize the touched player for one second. If they kill someone, they will come back in the game. When walls are closing, players in karts will disappears and cannot play anymore."

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Keep playing from a perimeter kart (Priority: P1)

An eliminated participant in Normal or Team mode stays involved by appearing in a small kart outside the board. They travel around its perimeter with left/right and throw bombs inward with their existing bomb action. The kart follows the corners without needing up/down steering.

**Why this priority**: This is the playable second chance that replaces waiting after elimination.

**Independent Test**: Eliminate a participant while at least two opposing sides remain alive before wall closing, drive their kart around the entire board in both directions, and throw a bomb inward from each edge.

**Acceptance Scenarios**:

1. **Given** an ongoing Normal or Team round before wall closing, **When** a participant is eliminated and the round does not end, **Then** they appear in a kart outside the nearest board edge, retaining their identity and team.
2. **Given** a kart, **When** the participant holds right or left, **Then** it moves clockwise or counterclockwise respectively, follows every corner, and never enters a playable square; releasing movement stops it.
3. **Given** a kart, **When** up, down, punch, or ordinary on-board movement is requested, **Then** the kart does not leave its track or perform those actions; simultaneous left/right leaves it stationary.
4. **Given** a kart beside each of the four board edges, **When** the participant uses the bomb action with capacity available, **Then** a bomb is thrown perpendicular to that edge toward the board, initially targeting the second playable square inward.
5. **Given** a Chaser round, **When** a runner or the chaser is eliminated, **Then** no kart appears and the existing Chaser outcome rules remain unchanged.
6. **Given** a kart outside the board, **When** on-board explosions occur or players collect items, **Then** the kart cannot be hit by those explosions, collect items, block players, or count as a living survivor.

---

### User Story 2 - Disrupt survivors with thrown bombs (Priority: P1)

A kart participant aims inward and throws a bomb over occupied squares. A survivor struck at a candidate landing square is immobilized for one second, while the bomb continues to the next available square like a punched bomb.

**Why this priority**: Throwing and its predictable targeting make perimeter participation useful and give it the requested distinct combat effect.

**Independent Test**: Throw into prepared lines containing empty squares, walls, destructible blocks, bombs, and survivors; check the landing square and the duration of every direct-hit immobilization.

**Acceptance Scenarios**:

1. **Given** a free second square inward from the kart, **When** a bomb is thrown, **Then** it lands there rather than on the first square.
2. **Given** the target square contains a wall, destructible block, bomb, or living player, **When** the thrown bomb reaches it, **Then** it skips that square and continues inward to the first free landing square, skipping any further occupied squares.
3. **Given** a living opponent occupies a candidate landing square, **When** the thrown bomb reaches that square, **Then** the opponent cannot move for one second, the impact itself does not eliminate them, and the bomb continues rather than occupying their square.
4. **Given** a survivor is immobilized, **When** they use bomb or punch actions, **Then** those actions remain available under their normal rules; another direct hit restarts one second of immobilization instead of adding durations.
5. **Given** no free landing square remains inward, **When** the bomb reaches the far edge, **Then** it searches back along the same line for the nearest free square, as a punched bomb does; if none exists, the throw is refused without consuming bomb capacity.
6. **Given** a kart bomb is in flight or has landed, **When** its fuse expires or it participates in a chain reaction, **Then** existing punched-bomb flight, fuse, explosion, and ownership rules apply, including explosions in flight and chain reactions after landing.
7. **Given** Team mode, **When** a throw strikes a teammate, **Then** direct-hit immobilization applies only with friendly fire enabled, and explosion damage follows the existing friendly-fire setting in either case.

---

### User Story 3 - Earn a return to the board (Priority: P2)

A kart participant who eliminates an opponent with their thrown bomb returns as a living player in the same round, with the same identity and team. They can be eliminated and earn another return while the opportunity remains open.

**Why this priority**: Revival turns a successful revenge attack into a meaningful second chance rather than only interference.

**Independent Test**: Credit an opponent elimination to a kart's thrown bomb, observe one return to the board, and repeat with a nonqualifying kill and with a second elimination of the revived participant.

**Acceptance Scenarios**:

1. **Given** a kart participant before wall closing, **When** an opponent elimination is credited to their kart-thrown bomb, **Then** their kart disappears and they return once as a living player, at the defeated opponent's square if safe or the nearest safe free square otherwise.
2. **Given** a revival, **When** the participant returns, **Then** their identity and team remain unchanged, their powers and bomb capacity reset to the round-start defaults, and no immobilization or temporary malus carries over.
3. **Given** one kart bomb eliminates several opponents together, **When** eliminations are resolved, **Then** its owner receives at most one revival; simultaneous qualifying owners each receive at most one non-overlapping return.
4. **Given** a kart participant, **When** their teammate is eliminated, another participant receives the kill credit, or one of their pre-elimination on-board bombs earns a kill, **Then** that event does not grant them a revival.
5. **Given** a revival earned while no square is safe and free, **When** the round continues before wall closing, **Then** the participant stays in the kart with one pending return and returns at the first opportunity a safe free square becomes available.
6. **Given** a revived participant before wall closing, **When** they are eliminated again without ending the round, **Then** they receive a kart again under the same rules.
7. **Given** an elimination that would leave one surviving player or team, **When** a kart owner earns a return in that same moment, **Then** eligible returns are resolved before the winner is determined; karts without an earned return never postpone the existing win or draw result.

---

### User Story 4 - Close the second-chance window consistently (Priority: P1)

When wall closing starts, every kart disappears and eliminated participants can no longer act or return during that round. Players and observers can distinguish living players, kart participants, and permanently eliminated participants in both the browser and terminal.

**Why this priority**: The cutoff is an explicit boundary of the feature and preserves the final showdown.

**Independent Test**: Start wall closing while karts and thrown bombs exist, attempt further movement, throws, and revenge kills, then start a rematch and verify all temporary state has cleared.

**Acceptance Scenarios**:

1. **Given** active karts, **When** the first wall-closing warning begins, **Then** all karts disappear and pending returns are cancelled before any kart movement, throw, or revival at that same moment.
2. **Given** wall closing has started, **When** a living participant is eliminated or a previously thrown kart bomb eliminates an opponent, **Then** no kart or revival is granted; already thrown bombs otherwise finish normally.
3. **Given** a returned living player, **When** wall closing begins, **Then** they stay on the board and play under the ordinary closing-wall rules.
4. **Given** a round ends or a rematch begins, **When** participants see the result or new board, **Then** no kart, pending return, or direct-hit immobilization from the previous round remains.
5. **Given** the same match viewed in the browser and terminal, **When** a participant enters a kart, is immobilized, revives, or loses their kart at wall closing, **Then** both views show the same state and retain that participant's identity and team indication.
6. **Given** a bot owns a kart, **When** an opponent can be targeted, **Then** the bot can navigate the perimeter and throw using the same allowed actions and information as a human; a bot taking over a departed participant retains their current kart or living state.

### Edge Cases

- Elimination at the wall-closing boundary: wall closing takes precedence, so no kart is created and no return is awarded.
- Last opponent or last opposing team eliminated: resolve already earned returns, then apply the existing result immediately; karts alone do not keep a round alive.
- Two participants become karts at the same track position: karts may overlap and pass without blocking each other or changing aim.
- Corners and equal-distance placement: use the clockwise edge order starting with the top edge to break ties; each corner belongs to the edge that begins there in clockwise travel, making inward throws unambiguous.
- A player moves into a prospective landing square while a bomb is flying: occupancy is checked on arrival, so they are hit and the bomb continues; a player who has left it is not hit.
- More than one survivor is encountered along a throw: each eligible survivor reached at a candidate landing square receives the same one-second immobilization.
- No safe square for revival: keep one pending return, never overlap a survivor or place a player into an active flame, and cancel it at closing or round end.
- A return square tie or several returns together: use top-to-bottom, left-to-right square order and ascending participant slot order to choose distinct safe squares reproducibly.
- A returned participant still owns thrown bombs: those bombs retain ownership and occupy their normal capacity until removed; no additional simultaneous bombs are granted by the return.
- A disconnected participant is replaced by a bot: transfer their kart position, thrown bombs, pending return, and immobilization as applicable rather than awarding a fresh life.
- A single-participant practice round ends on their elimination under the existing rules; karts do not create an opponent or delay that result.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: Normal and Team mode MUST grant eliminated participants a perimeter kart while the round continues before wall closing; Chaser mode MUST never grant karts.
- **FR-002**: A kart MUST be a small, identifiable vehicle outside the playable board, placed on the nearest perimeter position to the elimination square, preserving participant identity and team. The player MUST be visibly seated in the kart, retaining their current colour and character customization.
- **FR-003**: Left/right MUST move a kart counterclockwise/clockwise around a continuous track with automatic corner following; no movement input or conflicting left/right MUST stop it, and up/down and punch MUST have no kart effect.
- **FR-004**: A kart MUST face inward for throwing independently of travel direction, MUST never enter or obstruct the board, and MUST be unaffected by on-board explosions or item pickups; karts MUST NOT obstruct each other.
- **FR-005**: The existing bomb action MUST throw inward toward the second playable square from the adjacent board edge. Kart participants MUST have one kart-thrown bomb active at a time, with the ordinary round-start bomb fuse and blast range; pre-elimination bombs MUST retain their existing behavior and ownership without preventing this one kart bomb.
- **FR-006**: A thrown bomb MUST skip walls, destructible blocks, bombs, and living players and seek the next free square along the same inward line. At the far edge it MUST search backward for the nearest free square; with no free square anywhere on that line, the throw MUST have no effect or capacity cost.
- **FR-007**: A living player reached at a candidate landing square MUST be immobilized for exactly one second of game time, subject to Team friendly fire. The impact MUST NOT itself cause elimination, the bomb MUST continue past them, and a repeated hit MUST restart rather than accumulate immobilization.
- **FR-008**: Immobilization MUST prevent movement, including movement from a held direction, but MUST NOT disable otherwise permitted bomb or punch actions; movement MUST become available when the second expires and the effect MUST clear on elimination.
- **FR-009**: Kart-thrown bombs MUST follow existing punched-bomb rules after launch for travel, occupancy checks, fuse countdown, airborne explosions, landing, chain reactions, and kill attribution; Team explosion damage MUST respect existing friendly-fire rules.
- **FR-010**: Before wall closing, an opponent elimination credited under existing ownership and chain-reaction rules to a bomb thrown by a participant during their current kart stint MUST earn that participant one return. Teammate kills, other owners' kills, and bombs placed before entering that kart MUST NOT qualify.
- **FR-011**: A qualifying participant MUST leave their kart and return to the defeated opponent's square when safe and free, or the nearest safe free playable square by square distance otherwise. Safe means no wall, destructible block, bomb, living player, or active flame; equal choices MUST use the stated square order.
- **FR-012**: If no return square is available, the participant MUST retain one pending return while remaining in their kart; the return MUST occur as soon as a square becomes available before closing or round end, and additional kills MUST NOT stockpile returns.
- **FR-013**: A return MUST retain identity and team, reset powers and temporary effects to round-start defaults, and retain ownership of outstanding bombs, which count against ordinary on-board bomb capacity after return. Repeated elimination and earned return MUST remain possible before closing.
- **FR-014**: Eliminations and eligible returns at the same moment MUST be settled before applying existing Normal and Team victory rules. Kart participants and pending returns without an available return square MUST NOT count as living survivors or delay a result.
- **FR-015**: The beginning of the first wall-closing warning MUST remove all karts and cancel pending returns, taking precedence over kart actions and returns at that same moment. Later eliminations MUST NOT create karts, and later kills MUST NOT revive eliminated participants during that round.
- **FR-016**: Bombs already thrown when closing begins MUST finish under normal bomb rules without granting subsequent returns; players already returned before closing MUST remain ordinary living players.
- **FR-017**: Round end and rematch MUST clear all kart, pending-return, and immobilization state. A replacement bot MUST inherit the replaced participant's current state rather than receive another life.
- **FR-018**: Humans and bots MUST have the same kart abilities and restrictions. Bots MUST be able to navigate, aim inward, throw at opponents, and return without information unavailable to human participants; bots MUST NOT deliberately target teammates.
- **FR-019**: Browser and terminal participants, including local and networked players, MUST be able to use existing left/right and bomb bindings for kart actions, see kart identity and team, distinguish immobilization and revival, and see when kart participation ends. Existing supported keyboard, controller, and touch inputs MUST retain equivalent actions.
- **FR-020**: The same starting situation and ordered participant actions MUST produce the same kart positions, landings, immobilization durations, kill attribution, returns, and result. All supported board sizes MUST provide a continuous perimeter track and inward targeting from every edge.
- **FR-021**: Player-facing rules and controls documentation MUST describe eligibility, perimeter movement, throwing, immobilization, qualifying kills, revival, and the wall-closing cutoff. This feature MUST NOT add a new game mode, room setting, power-up, or changed Chaser rule.

### Key Entities *(include if feature involves data)*

- **Participation state**: Living on the board, eliminated in a kart, or eliminated without further actions; includes identity and team throughout a round.
- **Perimeter kart**: An eliminated participant's vehicle, position around the board, current travel direction, inward aim, and allowed throw capacity.
- **Kart-thrown bomb**: A bomb associated with its throwing participant and kart stint, including flight, landing, fuse, blast range, and qualifying kill ownership.
- **Direct-hit immobilization**: The affected survivor and the one-second movement restriction, refreshed by subsequent eligible hits and cleared on elimination.
- **Earned return**: A single revival entitlement from an opponent kill, with a preferred return location and eligibility that expires at closing or round end.
- **Wall-closing boundary**: The moment the first closing warning begins, after which kart participation and revival are unavailable for the remainder of the round.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: In every acceptance case for an ongoing pre-closing Normal or Team round, an eliminated participant becomes an identifiable kart by the next visible game update; no Chaser elimination creates one.
- **SC-002**: On all three supported board sizes, a kart can complete one uninterrupted circuit in each direction and throw inward from all four edges without entering the board, becoming stuck at a corner, or changing the board layout.
- **SC-003**: In every prepared targeting case covering free squares, each obstacle type, moving survivors, multiple survivors, the far edge, and a fully blocked line, the bomb lands or is refused exactly as specified; every eligible direct hit prevents movement for one second, within one game update of timing precision.
- **SC-004**: Every prepared qualifying-kill case grants at most one return per kart owner, and every teammate-kill, wrong-owner, pre-elimination-bomb, and post-closing case grants none. Returned participants can resume ordinary movement and bomb actions under their available capacity.
- **SC-005**: At wall closing, 100% of active karts disappear by the next visible game update, no eliminated participant can act or revive afterward, and already returned living players remain playable.
- **SC-006**: All acceptance scenarios produce matching participation states, landings, immobilization, returns, and round outcomes in the browser and terminal, for human and bot-controlled slots; repeating the same starting situation and actions produces the same result.
- **SC-007**: In a five-person playtest after reading the controls, at least four participants can independently drive around a corner, throw inward, and correctly explain how to return and when kart participation ends within two minutes of their first elimination.

## Assumptions

- This feature builds on the current Normal and Team modes, existing bomb ownership and punch behavior, closing-wall warnings, participant replacement, and supported input methods. It does not change board layouts, team membership, maximum participant count, or existing winner/draw conditions beyond resolving earned returns before the result.
- "Before and until the walls are closing" means strictly before the first warning of the closing pattern, not before the first block lands. Kart participation is automatic in eligible modes, not an optional room setting.
- Left/right means movement around the perimeter, not screen-relative movement on individual edges: right is clockwise and left is counterclockwise. Movement is not automatic when inputs are released. Kart travel uses the ordinary round-start player movement pace, unaffected by previously collected speed powers or maluses.
- "In front" means inward from the current edge, rather than along the kart's direction of travel. The first playable square adjacent to that edge is square one and the next is square two; border walls outside playable squares do not count toward this distance.
- A "free" bomb landing square excludes walls, destructible blocks, living players, and other landed bombs; pickups do not block landing, and airborne bombs follow the existing punch occupancy rules. A survivor skipped before reaching the initial second-square target is not struck.
- Team friendly fire governs direct-hit immobilization as well as explosion damage: disabling friendly fire protects teammates from both. Only opponent kills qualify for return, preventing teammate kills from rewarding a new life.
- Returning at the defeated opponent's location or the nearest safe alternative is the default second-chance behavior. There is no new protective shield or invulnerability on return; ordinary explosions can eliminate the player again. Square distance is the number of horizontal and vertical steps between squares, without considering obstacles.
- If a qualifying throw eliminates several opponents, the lowest-slot defeated opponent supplies the preferred return square; simultaneous return owners are processed in ascending slot order. Nearest perimeter placement and return ties use the fixed orders defined in Edge Cases.
- Bomb range, fuse, movement pace, and one-bomb kart capacity use the defaults above as initial balance choices. Pre-elimination powers are not retained in the kart or restored on return, and no additional kart-specific powers or audiovisual assets are prescribed by this specification.