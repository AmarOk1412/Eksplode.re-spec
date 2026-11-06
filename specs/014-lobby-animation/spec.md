# Feature Specification: Animated Lobby Scene

**Feature Branch**: `main` (unchanged; no branch-creation hook configured)

**Created**: 2026-11-06

**Status**: Draft

**Input**: User description: "Lobby should contain instead a logo a 2d side view animation to make it alive. Animation should be the player running from side view and dropping bombs. Other players appearing and running in the other way or same way. Sometimes the player may blew and another animation can start. It can also be the player in a kart going from one way to another."

**Planning refinement (2026-11-06)**: Replace the logo zone with an animated zone; randomly select running/bomb-drop, same-level or background runners, kart, and bomb-push mishap vignettes. The player confirmed that the target is the existing main-menu logo above Create new game / Join a game / Options, not the room lobby. Throughout this feature, "lobby scene" names that main-menu decoration.

## Clarifications

### Session 2026-11-06

- Q: Which screen receives the scene, given that the existing logo is on the main menu? A: Replace the main-menu logo; do not add a scene inside the room lobby.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - See a living lobby instead of a static logo (Priority: P1)

As a player on the main menu, I see a side-view character running and dropping bombs in the space previously occupied by the logo, making the menu feel like part of the game's world.

**Why this priority**: Replacing the static logo with recognizable animated activity is the central requested experience.

**Independent Test**: Open the main menu without interacting with the scene and observe a complete running-and-bomb sequence, followed by another randomly selected sequence.

**Acceptance Scenarios**:

1. **Given** normal motion is enabled, **When** the main menu opens, **Then** a two-dimensional side-view scene replaces the entire logo zone and begins playing automatically without a click or a joined opponent.
2. **Given** a running sequence is playing, **When** a character crosses the scene, **Then** its running motion and facing direction match its travel and it visibly leaves at least one bomb behind.
3. **Given** a bomb has been dropped, **When** its fuse completes, **Then** it visibly explodes and its effect clears before the next sequence begins.
4. **Given** a sequence ends, **When** its characters exit or its action resolves, **Then** a new sequence begins within two seconds rather than leaving a permanently empty scene.
5. **Given** successive sequences, **When** new sequences are selected, **Then** their ordering is randomized rather than following one fixed playlist, while every required action remains observable within the specified variety interval.

---

### User Story 2 - Continue using a comfortable, accessible menu (Priority: P1)

As a player preparing a match, I can use Create new game, Join a game, and Options while the scene plays without an extra scene control. My reduced-motion preference stops animation when requested.

**Why this priority**: Decorative activity must not obstruct joining friends, configuring the room, or starting a match.

**Independent Test**: Complete the existing main-menu actions with supported inputs, verify no scene-specific button or focus stop is added, and repeat with reduced motion enabled.

**Acceptance Scenarios**:

1. **Given** the scene is playing, **When** a player uses a main-menu control, **Then** the action remains available and neither moving characters nor explosions cover the control or take its input or focus.
2. **Given** a player has requested reduced motion, **When** the main menu appears or that preference changes while it is open, **Then** the scene shows a still representative image without running, travelling karts, or animated explosions.
3. **Given** the main menu, **When** the player views or navigates its controls, **Then** no scene pause/play icon, button, or additional focus target is present.
4. **Given** a narrow or wide display, **When** the main menu is shown or resized, **Then** the scene remains within its reserved area, does not cause horizontal scrolling, and does not overlap or shift adjacent text or controls as the action changes.
5. **Given** animation cannot be displayed, **When** the main menu opens, **Then** a still game-themed fallback occupies the scene area and every menu control remains usable.
6. **Given** an animated main menu, **When** the player navigates away or hides the page, **Then** the scene stops; returning starts at most one scene without catching up hidden elapsed time.

---

### User Story 3 - Watch playful encounters with other characters (Priority: P2)

As a waiting player, I see additional characters enter, run alongside or against the lead character's direction, and occasionally get caught in a bomb explosion before a new vignette begins.

**Why this priority**: Encounters and changing outcomes provide the requested variety beyond a single repeated run.

**Independent Test**: Observe the encounter variants and verify same-direction travel, opposite-direction travel, and an explosion ending independently of the actual room roster.

**Acceptance Scenarios**:

1. **Given** a lead character is running, **When** a same-direction encounter plays, **Then** at least one visually distinguishable additional character enters and runs in the same direction.
2. **Given** a lead character is running, **When** an opposite-direction encounter plays, **Then** at least one visually distinguishable additional character enters and runs in the opposite direction.
3. **Given** a character is within a depicted bomb explosion, **When** an explosion-ending variant plays, **Then** that character is visibly caught in a brief, non-graphic cartoon elimination and another sequence starts within two seconds of the effect ending.
4. **Given** multiple sequences play, **When** their outcomes are observed, **Then** some characters exit safely and some sequences end with an elimination; an elimination is not required on every pass.
5. **Given** encounter variants, **When** their runners pass, **Then** some share the lead character's ground level and some appear smaller on a visibly separate background level behind the foreground action.
6. **Given** a bomb-push mishap variant, **When** a character visibly pushes a bomb, **Then** another bomb enters, briefly flattens that character in a non-graphic cartoon gag, and the scene clears before another sequence begins.

---

### User Story 4 - See kart drive-by variations (Priority: P2)

As a waiting player, I sometimes see a character riding a kart across the scene, providing a different kind of movement between running encounters.

**Why this priority**: Kart passes are an explicitly requested animation variant and add variety without changing lobby behavior.

**Independent Test**: Observe a kart pass in each direction and verify each finishes cleanly before another vignette begins.

**Acceptance Scenarios**:

1. **Given** a kart variant starts, **When** the character crosses the scene, **Then** it is visibly riding a kart shown from the side rather than running on foot.
2. **Given** the available kart variants, **When** leftward and rightward passes are observed, **Then** each enters from one side, travels across the scene facing its direction of travel, and exits on the other side.
3. **Given** a kart has exited, **When** the vignette ends, **Then** another vignette starts within two seconds with no leftover kart, bomb, or explosion from the previous sequence.

### Edge Cases

- An empty room or a room with only one human still shows multi-character encounters; decorative characters do not imply real occupants.
- Players joining, leaving, changing readiness, or changing room settings do not reset or alter real room state because of the scene.
- Rapid lobby entry and exit, repeated resizing, or returning after an inactive period must not produce overlapping copies of the scene or a burst of accumulated explosions.
- Opening Options or navigating to another screen must leave that screen readable, focused, and operable, with main-menu animation stopped while hidden.
- Switching reduced motion on during an explosion immediately replaces animated activity with a still image; switching it off resumes only when the main menu is visible and uncovered.
- A failed animation must not leave the lobby waiting for the scene before controls become usable.
- Explosions must not use full-screen flashes or flash more than three times in any one-second period.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: The web main menu MUST remove its existing logo zone and replace it with an automatically playing two-dimensional side-view animated zone when motion is allowed, while keeping the game name identifiable. The room lobby MUST remain unchanged.
- **FR-002**: The scene MUST include a character running across its visible area with a matching facing direction and visibly dropping bombs behind it.
- **FR-003**: Dropped bombs MUST show a readable fuse-to-explosion progression and clear their effects before the next vignette begins.
- **FR-004**: The scene MUST include encounters with at least one additional visually distinguishable character travelling in the lead character's direction and encounters travelling in the opposite direction, with variants on the same ground level and on a visibly separate, smaller background level behind the foreground action.
- **FR-005**: The scene MUST include both safe exits and occasional non-graphic cartoon eliminations caused by depicted bomb explosions, including a variant where the lead character is caught.
- **FR-006**: The scene MUST include side-view kart passes in both leftward and rightward directions, with the rider facing the direction of travel.
- **FR-007**: Playback MUST randomly select successive vignettes without user action rather than use a fixed playlist, with no transition leaving an empty scene for longer than two seconds. Each uninterrupted 90-second playback period MUST include a running bomb-drop, both encounter directions, same-level and background runners, a lead-character elimination, a safe exit, kart travel in both directions, and a bomb-push/flattening mishap.
- **FR-008**: All depicted actors, bombs, eliminations, and karts MUST be decorative and MUST NOT change room membership, readiness, settings, match statistics, or gameplay. Playback MUST work without other players being present.
- **FR-009**: The scene MUST stay inside a stable reserved main-menu area across supported desktop and mobile layouts, without covering controls, obscuring text, creating horizontal scrolling, or shifting nearby content between variants.
- **FR-010**: The scene MUST NOT intercept menu input, take focus automatically, or add decorative actors to focus navigation or screen-reader announcements. The scene MUST NOT expose a pause/play icon, button, or additional focus target.
- **FR-011**: A reduced-motion preference MUST produce a still representative scene instead of animation and MUST take effect when changed while the main menu is open. Removing that preference resumes playback only when visibility permits.
- **FR-012**: The scene MUST remain silent and MUST avoid full-screen flashes and flashing above three times per second.
- **FR-013**: Scene playback MUST stop when the main menu is not visible; returning MUST show only one scene, without replaying a burst of activity accumulated while hidden.
- **FR-014**: If animation is unavailable, the main menu MUST show a still game-themed fallback without blocking existing menu actions.
- **FR-015**: The scene MUST include a character visibly pushing a bomb before being briefly flattened by another incoming bomb, using a non-graphic cartoon effect and clearing the outcome before the next vignette.

### Key Entities

- **Lobby Vignette**: A randomly selected short decorative sequence with a running, kart, or bomb-push premise, travel directions, depth levels, optional encounters, and a safe-exit, explosion, or flattening ending; followed by another vignette.
- **Decorative Character**: A fictional game character with a distinguishable appearance, direction of travel, depth level, and running, riding, pushing, flattened, or eliminated presentation; not an actual room occupant.
- **Decorative Bomb**: A visual object dropped during a vignette, progressing through a fuse, explosion, and cleared state without affecting a match.
- **Motion Preference**: The player's reduced-motion preference, determining whether the lobby scene moves or remains still.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: With normal motion enabled, players see a moving side-view character within one second of the main menu becoming usable, with no action required to start playback.
- **SC-002**: During any uninterrupted 90-second observation, players can see every required action from FR-007, with zero empty transitions longer than two seconds and no leftover objects between vignettes.
- **SC-003**: In a five-minute main-menu session, 100% of tested existing menu actions remain completable with their supported input methods, with no scene-caused focus changes, blocked controls, or changes to match or room state.
- **SC-004**: At supported desktop and mobile sizes, including the narrowest supported menu width, every tested variant has zero overlaps with menu controls or text and causes no additional horizontal scrolling.
- **SC-005**: In all reduced-motion, hidden-menu, and animation-failure checks, players retain usable menu controls; reduced-motion and hidden-menu checks show no continuing animated movement after the preference or visibility change takes effect.
- **SC-006**: In a review with five players, at least four correctly identify the scene as a side-view running/bomb and kart animation and describe it as making the menu feel more alive without preventing them from finding Create new game, Join a game, and Options.

## Assumptions

- Scope is the existing web main-menu logo zone only, as confirmed during planning. Room lobbies, the room browser, scoreboard, gameplay view, and terminal interface are unchanged.
- "The player" means a recognizable decorative game character, not necessarily the viewer's saved appearance. Matching actual participants, names, cosmetics, or live inputs is out of scope.
- The scene uses the game's established character, bomb, explosion, and kart identity. It is an ambient cartoon vignette, not a playable tutorial or a demonstration that must enforce match rules.
- Both running and kart sequences belong in this feature; they alternate over time rather than making kart playback a room-mode setting.
- Exact choreography and art treatment are design decisions; sequence ordering is randomized while meeting the required actions and 90-second variety target.
- "Smashed by another" is interpreted as another bomb entering and briefly flattening the character that pushed the first bomb; no graphic injury or new match mechanic is introduced.
- No new sound effects, music, persistent player data, gameplay settings, or scene pause/play controls are required. The explicit user revision removes the previously planned scene control; reduced-motion and automatic visibility handling remain.
- The existing lobby actions, supported input methods, responsive layouts, and accessible motion preferences are dependencies to preserve, not replace.