# Research: Bomberman Multiplayer Game

All Technical Context items are resolved; no NEEDS CLARIFICATION remain.

## R1. Repository layout
- **Decision**: Code in `game/`, a standalone git repo; Spec Kit artifacts remain in the outer workspace.
- **Rationale**: Requested by the user; keeps game history clean and publishable on its own.
- **Alternatives**: Single repo at the workspace root (mixes tooling and game history).

## R2. Simulation model
- **Decision**: Fixed 20 Hz tick, pure `step(state, inputs) -> events`, own SplitMix64 RNG seeded per match.
- **Rationale**: Determinism (FR-017), trivial unit tests, simple network sync, no `rand` dependency.
- **Alternatives**: Real-time continuous movement (non-deterministic, hard to test); `rand` crate (extra dep, version-dependent streams).

## R3. Movement and conflicts
- **Decision**: Tile-based moves with cooldown; speed power-up lowers cooldown; simultaneous entry into the same tile is resolved by the order `(tick + index) % 4` so no slot is permanently favoured.
- **Rationale**: Matches "one tile at a time" spec; deterministic and fair.
- **Alternatives**: Sub-tile pixel movement (complex collisions); fixed index order (always favours player 1).

## R4. Bots
- **Decision**: One default bot: build a danger map from bombs/flames (with chain reactions), BFS to nearest safe tile, place bombs next to blocks/opponents only when an escape path exists, flee when in danger. Takes an `Observation`, returns a `PlayerInput`.
- **Rationale**: Meets FR-009, SC-004; simple to unit test with scripted arenas.
- **Alternatives**: Minimax/MCTS (heavy, overkill); random bot (fails SC-004).

## R5. CLI rendering and input
- **Decision**: `ratatui` with `crossterm`. Rendering tested with `TestBackend`. Shared keyboard: P1 arrows + Space, P2 WASD + Left Shift/E, P3 IJKL + O, P4 numpad/other; `q` quits. Terminals deliver no key-release, so input is "last pressed direction this tick".
- **Rationale**: Mature, testable; avoids hand-written terminal escape code.
- **Alternatives**: Raw `crossterm` only (more drawing code, harder to test).
- **Note**: Four humans on one keyboard is awkward; the default local setup is 1 human + bots, but all 4 key sets are supported.

## R6. Networking
- **Decision**: `std::net::TcpStream`, newline-delimited JSON (`serde_json`), one reader thread per connection, `mpsc` channels, host authoritative. Default port 7878. Host sends full snapshot each tick (state is small: 15x13 grid + few entities).
- **Rationale**: No async runtime needed, debuggable on the wire, adequate for 4 players on LAN (SC-006).
- **Alternatives**: `tokio` (extra complexity); UDP (loss handling needless at this scale); delta/rollback netcode (not needed).
- **Security**: Host validates every message (size limit 64 KiB per line, known variants, slot ownership, rate); malformed data yields an error reply and never panics. No encryption (out of scope per spec).

## R7. Disconnects
- **Decision**: On disconnect the host notifies others and switches the slot to a bot (setting configurable to "remove player"); the match continues. Host leaving ends the match with a message.

## R8. Isometric GUI
- **Decision**: `macroquad`, 2D polygons. Projection `sx = (x - y) * TW/2`, `sy = (x + y) * TH/2 - z * TH`; draw order by `(x + y)` then height. Cubes = top + left + right faces with brightness 1.0 / 0.8 / 0.6 of the player colour. Palette: red, blue, green, yellow (distinguishable, plus a number badge for colour-blind use). Interpolate positions between ticks for smoothness.
- **Rationale**: Few dependencies, fast to build, projection/depth logic is pure and testable.
- **Alternatives**: `bevy` (heavy compile, large API surface); macroquad 3D cubes with an orthographic camera (less control over the look, harder to unit test).

## R9. Testing strategy
- **Decision**: Test pyramid: (a) `core` unit tests per rule; (b) scenario tests each citing a spec acceptance id (e.g. `us1_as3_blast_stops_at_wall`); (c) property-style tests (seeded loops) for determinism and no-panic on random inputs; (d) bot scripted scenarios + 100-match soak test (SC-003); (e) loopback network tests for join/full/disconnect/invalid message; (f) CLI `TestBackend` snapshot tests; (g) GUI pure projection/depth tests and a headless smoke run where available.
- **Rationale**: Constitution II and FR-019.

## R10. Commit convention
- **Decision**: `part: short desc` (subject <= 80 chars) + wrapped body; enforced by a `commit-msg` hook installed with `git config core.hooksPath .githooks`.
- **Rationale**: Requested by the user.
