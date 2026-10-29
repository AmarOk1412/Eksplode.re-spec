# Feature Specification: Multi-Room Servers and Discovery

**Feature Branch**: `main` (no feature branch requested)

**Created**: 2026-10-28

**Status**: Draft

**Input**: Host several independent rooms simultaneously; let players create or join rooms, configure name, capacity, game mode and password, and rejoin after disconnection. List and join rooms on known servers. Preserve earlier mobile and keyboard navigation. Provide owner permissions, room lifecycle management and significant-event logging for operators.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Choose a room and play independently (Priority: P1)

A player browses available rooms, creates a room or joins one, and plays without affecting other rooms. Different rooms can run Normal, Team and Chaser at the same time.

**Why this priority**: Independent games are the core capability.

**Independent Test**: Run three rooms with different modes and verify that starting, playing and ending one never changes the others.

**Acceptance Scenarios**:

1. **Given** a player outside a room, **When** they connect, **Then** they can list rooms, create one or select one to join, without being silently enrolled in a match.
2. **Given** three rooms using different supported modes, **When** each starts and receives player actions, **Then** only its members receive its gameplay state and each match follows its own settings, players, bots and result.
3. **Given** simultaneous attempts to take the final slot, **When** the server decides admission, **Then** exactly one succeeds and the other receives a full-room message without losing their current membership.
4. **Given** an ongoing match, **When** a new player attempts a fresh join, **Then** admission is refused as in-progress; a returning participant can use the reconnect flow instead.

---

### User Story 2 - Own and manage a room (Priority: P1)

The creator owns a room, chooses who may enter through password protection or removal, and controls settings and match start. Other players can ready themselves, play and leave, but cannot administer the room.

**Why this priority**: Separate rooms need clear authority and admission rules.

**Independent Test**: Create a protected room, compare owner and member actions, and verify admission, settings validation and ownership transfer.

**Acceptance Scenarios**:

1. **Given** creation, **When** the creator confirms valid settings, **Then** they become owner and members see the name, capacity, mode, board size, round time and relevant friendly-fire choice.
2. **Given** a protected room, **When** a player enters a missing or incorrect password, **Then** admission fails without changing membership or revealing the password; the correct password admits them if capacity and phase permit.
3. **Given** the lobby, **When** the owner changes valid settings, **Then** all members see the changes; unauthorized or invalid changes leave the room unchanged with a clear reason.
4. **Given** an active match, **When** settings changes are requested, **Then** they are refused; the lobby retains the settings for the next round.
5. **Given** a room owner, **When** they remove another participant, **Then** that participant leaves the room, their reconnect entitlement is revoked, and no other room is affected.
6. **Given** an owner who explicitly leaves or exhausts their reconnect window, **When** connected humans remain, **Then** ownership passes to the longest-present connected human, with ties resolved by slot order.

---

### User Story 3 - Recover from a lost connection (Priority: P2)

A briefly disconnected player can reclaim the same participant in their previous room, without restarting its match or taking someone else's place.

**Why this priority**: Network interruptions should not unnecessarily end a session.

**Independent Test**: Disconnect a member during a match, reconnect before and after the reservation expires, and attempt an unauthorized reclaim.

**Acceptance Scenarios**:

1. **Given** an unexpected disconnection, **When** fewer than 60 seconds have elapsed, **Then** the participant's slot remains reserved, a bot controls it during play, and a valid return restores the participant's current state rather than rewinding the match.
2. **Given** a reserved slot, **When** another person presents the same display name without proof of the previous session, **Then** they cannot reclaim it.
3. **Given** an expired reservation, explicit leave, removal or closed room, **When** rejoin is attempted, **Then** the player receives a clear reason and can return to room selection; they are not silently assigned another identity.
4. **Given** the last human unexpectedly disconnects, **When** reservations remain valid, **Then** the room remains recoverable until their deadlines; when none remain it closes and releases its resources.

---

### User Story 4 - Discover and join known servers (Priority: P2)

A player sees rooms from their current server and operator-configured known servers, can identify the host, and can join a compatible remote room without confusing identically named rooms.

**Why this priority**: Discovery makes independently hosted games accessible without distributing one match's authority.

**Independent Test**: Start two known servers, publish identically named rooms, join a remote room, and make one server unavailable.

**Acceptance Scenarios**:

1. **Given** reachable known servers, **When** room discovery is refreshed, **Then** compatible rooms show hosting server, name, mode, occupancy, capacity, phase and password-required status, but never passwords or rejoin secrets.
2. **Given** a remote listing, **When** the player joins, **Then** the destination host revalidates admission and becomes the sole authority for that room; selection alone does not discard an existing membership.
3. **Given** an unreachable or incompatible known server, **When** discovery runs, **Then** it reports that source as unavailable and local rooms remain usable; stale entries cannot admit players without host confirmation.

---

### User Story 5 - Navigate consistently and diagnose events (Priority: P2)

Players use room selection, setup, passwords, management and reconnect screens with the existing supported input methods. Operators can follow a room's significant events without exposing secrets.

**Why this priority**: New workflows must remain accessible and diagnosable.

**Independent Test**: Complete room flows using keyboard only, controller and mobile touch, repeat terminal-supported flows, and inspect an operator event history.

**Acceptance Scenarios**:

1. **Given** each supported interface and input method, **When** a player lists, creates, joins, configures, leaves or rejoins a room, **Then** every applicable action is reachable without a mouse, focus is visible, dialogs restore focus, and mobile controls remain usable without obscuring gameplay.
2. **Given** creation, admission, rejection, departure, disconnect, rejoin, expiry, ownership transfer, settings change, match start/end, room closure or discovery failure, **When** the event occurs under normal operation, **Then** operators receive a timestamped record identifying its server, room when applicable, actor when applicable, event and outcome. During queue saturation or output failure, logging MUST NOT block gameplay; dropped records MUST be counted and reported when output recovers.
3. **Given** passwords or session proofs submitted during these flows, **When** records are inspected, **Then** none contains those secrets; routine movement and complete game-state dumps are not required significant events.

### Edge Cases

- Duplicate names on one or several servers remain distinguishable by stable room and host identities.
- Occupancy includes reserved human seats and bots; capacity never exceeds four and Team/Chaser require four participants to start.
- Capacity cannot be reduced below occupied or reserved seats; a mode requiring four rejects a lower capacity. Bots fill otherwise available slots using existing controls.
- Concurrent create, start, join, removal and settings requests produce one consistent decision per room, never partial changes or cross-room effects.
- An owner is temporarily absent: ownership is reserved during the reconnect window and no other member can exercise owner powers until transfer.
- A reclaim after elimination restores an eliminated participant, not a new life; after round end it restores lobby membership if the reservation remains valid.
- Two connections attempt to reclaim one reservation: only one controls the participant, and superseded credentials cannot be reused.
- Empty rooms with bots only close when no valid human reservation remains.
- A player disconnects and the last connected player explicitly leaves: the absent player's reservation remains valid; the room closes only after no connected humans or valid reservations remain.
- An admission reply times out: the player sees an unknown outcome until it is reconciled, rather than blindly submitting another admission. Expired or cancelled pending work cannot later admit them.
- A resume reply or its acknowledgment is lost: a bounded retry recovers the replacement proof without creating another controller or extending the original reservation.
- Invalid or oversized names, passwords and discovery results are refused safely. Repeated failed password/rejoin attempts are bounded without blocking unrelated rooms.
- Destination loss during a cross-server join leaves the player able to retry or return to selection; no claim of successful admission is shown.
- Server restart invalidates rooms and reconnect reservations; restart persistence and live migration are not promised.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: A server MUST host multiple independent rooms concurrently, including different existing game modes, with isolated membership, settings, state, bots, timing and results (Story 1).
- **FR-002**: Players MUST choose creation or joining from a room directory showing host, name, mode, occupancy, capacity, phase and protection status (Stories 1, 4).
- **FR-003**: Creation MUST establish a unique room identity and creator ownership; a connected player MUST belong to at most one room at a time (Stories 1, 2).
- **FR-004**: The host MUST decide all admission and gameplay actions, reject cross-room actions, and send gameplay state only to authorized members of the relevant room (Story 1).
- **FR-005**: Owners MUST configure name, capacity from one to four, existing mode, board size, round time, friendly fire where applicable and optional password; defaults remain those of feature 004 with capacity four and no password (Story 2).
- **FR-006**: Only the owner MUST change settings, manage bots, remove another participant or start a match; members retain their own ready, gameplay and leave actions (Story 2).
- **FR-007**: Settings MUST be validated together and published consistently to members; active matches MUST reject changes; start MUST enforce existing mode participant requirements (Story 2).
- **FR-008**: Admission MUST reject full, protected-with-invalid-password, in-progress, closed or unsupported rooms with a specific safe reason and no partial membership changes. Expired or cancelled pending admissions MUST NOT commit; timed-out outcomes MUST be reconciled before another admission is attempted (Stories 1, 2, 4).
- **FR-009**: Unexpected disconnections MUST reserve the same participant for 60 seconds; during play the existing bot takeover MUST preserve the participant's current role, team and state (Story 3).
- **FR-010**: Rejoin MUST require proof of the prior session, allow only one controlling connection, and restore current room state without replaying missed actions or reviving eliminated players. Loss of a proof-rotation reply or acknowledgment MUST permit bounded recovery of the same operation without extending its reservation (Story 3).
- **FR-011**: Explicit departure, owner removal, expiry or closure MUST invalidate rejoin entitlement; a room with no connected humans and no unexpired reservations MUST close and release its resources (Stories 2, 3).
- **FR-012**: Owner absence MUST reserve ownership until return or expiry; permanent departure MUST transfer ownership deterministically to an eligible connected human, or leave no administrator until an eligible human rejoins (Stories 2, 3).
- **FR-013**: Operators MUST configure known servers; players MUST list and join compatible rooms on those servers, with unambiguous host identity and destination admission checks (Story 4).
- **FR-014**: Remote discovery failures MUST NOT prevent local room creation, joining or play; stale listings MUST be identifiable and never substitute for destination admission (Story 4).
- **FR-015**: Room flows MUST retain web/terminal parity, keyboard-only and controller navigation from features 005-006, and responsive touch and gameplay controls from feature 007 (Story 5).
- **FR-016**: Under normal operation, operators MUST receive a record for every significant lifecycle, permission, membership, match-transition and discovery event in Story 5, with time, host, room/actor when applicable, event and outcome; secrets MUST be excluded. During queue saturation or output failure, logging MUST NOT block gameplay; dropped records MUST be counted and reported when output recovers (Story 5).
- **FR-017**: Invalid actions and exhausted room/server limits MUST fail safely with clear feedback, bounded work and no state change to unrelated rooms (Stories 1, 2, 4; Edge Cases).
- **FR-018**: Existing deterministic gameplay, supported modes, powers, audio, bots and four-participant limit MUST remain unchanged within each room (Stories 1, 5).
- **FR-019**: Clients and hosts MUST establish room-management compatibility before admission; unsupported versions MUST receive an explicit upgrade-required rejection instead of entering an unintended room (Stories 1, 4).

### Key Entities *(include if feature involves data)*

- **Server**: Stable host identity, supported capabilities, reachable connection addresses and configured known-server sources.
- **Room**: Host-scoped identity, owner, settings, lifecycle, memberships and independent current match.
- **Room Listing**: Public room summary plus source freshness and compatibility; contains no admission secrets.
- **Membership**: Participant identity, room, slot, join order, connected/absent status, permissions and optional reservation deadline.
- **Reconnect Entitlement**: Private proof authorizing reclamation of a specific reserved membership until expiry or revocation.
- **Significant Event**: Operator-visible occurrence with time, host, room and actor where applicable, event name, outcome and safe contextual details.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Three concurrent four-participant rooms, one per supported mode, complete 20 rounds each with zero cross-room state changes or gameplay disclosures.
- **SC-002**: At least 95% of room listings and confirmed joins complete within two seconds on a healthy local network; gameplay input feedback remains within the existing 200 ms 95th-percentile target at this load.
- **SC-003**: All tested unauthorized settings, removal, start, cross-room and invalid-password attempts leave protected state unchanged, including 100 simultaneous attempts at a final seat with exactly one admission.
- **SC-004**: All 20 prepared return scenarios, including lost resume replies and acknowledgments, correctly reclaim within the 60-second window or reject expired/revoked/forged returns without restarting matches, extending reservations through retries or duplicating control.
- **SC-005**: With two known servers, every local and remote room is distinguishable and joinable when eligible; loss of one source leaves healthy-source discovery usable within five seconds.
- **SC-006**: All room lifecycle scenarios release closed-room memberships and reservations, and at least 100 create/play/close cycles leave no joinable ghost rooms.
- **SC-007**: All primary room workflows pass on web keyboard-only, controller, mobile touch and terminal-supported controls, with no focus trap or obscured mobile gameplay controls.
- **SC-008**: Every significant-event category in Story 5 produces a correlated operator record with zero drops under ordinary three-room load and zero exposed passwords or rejoin proofs. Separate saturation/output-failure scenarios leave gameplay running and report the exact dropped-record count after output recovers.

## Assumptions

- This feature supersedes feature 004's single-room and immediate-last-disconnect closure restrictions, not its gameplay rules. Features 001-007 otherwise remain applicable.
- Feature 004's requirement to retain protocol version 1 compatibility is superseded by explicit room-capability negotiation and safe upgrade-required rejection; silently mapping legacy joins into arbitrary rooms is excluded.
- Multiple games means simultaneous independent matches of the existing modes, not a plug-in framework for unrelated games.
- Cross-server support includes listing and connecting to a known host now; one match spanning several authoritative servers, migration and server federation trust are future scope.
- A room's capacity counts all participant slots, including bots and reconnect reservations. Existing bot replacement rules continue where they do not consume reserved human slots.
- The reconnect window is 60 seconds, measured by the host; explicit departures and removals are not temporary failures. Rooms and entitlements need not survive server restarts.
- Owners control admission through passwords and removal, not arbitrary per-action permission editing or persistent bans. Active-play removal uses existing bot takeover.
- Known servers are configured by the operator, not arbitrary player-submitted addresses. Remote play requires clients to reach the destination host.
- Significant-event output to stdout is a planning constraint requested by the user; event format, safe fields and operational bounds are specified in the design contracts.