# Feature Specification: Players Pass Through Each Other

**Feature Branch**: `010-player-collision`

**Created**: 2026-10-29

**Status**: Draft

**Input**: User description: "Player doesn't collide with other players, they can go through"

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Move freely through other players (Priority: P1)

A player moves their character toward a tile currently occupied by another player (human or bot). Today the move is blocked as if the other player were a wall; instead, the move must succeed and both players end up able to occupy the same tile at the same time.

**Why this priority**: This is the entire scope of the request — restoring free movement around other players removes a frequent source of players getting stuck behind each other or being used to block narrow corridors, which does not match the intended Bomberman-style feel.

**Independent Test**: Place two players adjacent to each other in a corridor one tile wide (walls on both sides) and move one player into the other's tile. Verify the move succeeds, both players report the same position, and neither player's movement is rejected or reverted.

**Acceptance Scenarios**:

1. **Given** two players standing on adjacent tiles, **When** one player moves into the tile occupied by the other, **Then** the move succeeds and both players may occupy the same tile simultaneously.
2. **Given** a player attempts to move into a tile containing a wall, destructible block, or bomb, **When** the move is processed, **Then** the move is still blocked exactly as before — only player-vs-player blocking is removed.
3. **Given** a bomb explodes on a tile where two or more players are standing together, **When** the blast resolves, **Then** every player on that tile is affected by the blast exactly as if they were on separate tiles (no immunity or special handling is granted by sharing a tile).
4. **Given** any supported game mode (Normal, Team, Chaser, etc.), **When** players move around each other, **Then** players never block one another's movement in any mode.

---

### Edge Cases

- What happens when three or more players converge on the same single tile at once? All of them must be able to occupy it simultaneously; none is pushed out or blocked.
- What happens in team-based or chaser-based modes where a "catch" or "tag" interaction may depend on players sharing a tile? Sharing a tile must remain possible and must not be prevented by movement rules (any catch/tag logic itself is out of scope and unaffected by this change).
- What happens to bots pathfinding through tiles occupied by other players? Bots must treat player-occupied tiles as passable, the same as any other player.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: The game MUST allow a player to move onto a tile that is already occupied by one or more other players.
- **FR-002**: The game MUST continue to block player movement onto tiles occupied by walls, destructible blocks, and bombs, unchanged from current behavior.
- **FR-003**: The game MUST allow two or more players to occupy the same tile simultaneously, for any amount of time, without forcing a separation.
- **FR-004**: This movement behavior MUST apply identically across all game modes and to both human and bot-controlled players.
- **FR-005**: Removing player-vs-player blocking MUST NOT change any other rule (blast damage, scoring, power-up pickup, win conditions) — those rules continue to apply per-player regardless of whether a tile is shared.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: In a one-tile-wide corridor, a player can cross through a tile occupied by another player 100% of the time, with zero rejected or reverted moves attributable to the other player's presence.
- **SC-002**: No regression occurs in wall, destructible block, or bomb collision: those moves are blocked exactly as often as before this change (no change in blocked-move rate for non-player obstacles).

## Assumptions

- "Collide" refers only to movement blocking between players; it does not refer to bomb blast damage, power-up pickups, or any mode-specific tag/catch mechanic, all of which already operate per-player and are unaffected.
- This change applies to the core simulation rules (shared by CLI and web frontends per the project's dual-interface parity principle), not to a single frontend only.
- No visual change (e.g., sprite layering when players overlap) is in scope beyond what is needed to render overlapping players without crashing or glitching.
