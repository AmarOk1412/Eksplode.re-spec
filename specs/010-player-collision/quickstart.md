# Quickstart: Validating Player-Pass-Through

## Prerequisites

- `cargo build --workspace` succeeds.

## Unit tests (write failing first, per Principle II)

1. `crates/core/tests/movement.rs`: two players adjacent in a one-tile-wide corridor; player A moves into
   player B's tile; assert the move succeeds (`Event::Moved` emitted, `A.pos == B.pos` afterward), not
   blocked.
2. `crates/core/tests/movement.rs`: a player moving into a wall, a destructible block, and a landed bomb
   is still blocked exactly as before (regression guard for FR-002) — this should already exist; confirm
   it still passes unmodified.
3. `crates/core/tests/movement.rs` (or `round_end.rs`): three or more players converge on one tile across
   consecutive ticks; assert all end up with equal `pos` and none is pushed out.
4. `crates/core/tests/*` (wherever blast/elimination is tested, e.g. `chain.rs`): two players sharing a
   tile when a blast covers it are each eliminated independently (no immunity granted by sharing a tile) —
   regression guard for FR-003.
5. `crates/bots/tests/`: a bot's path-planning (`pathing::passable`/the search it feeds) treats a tile
   occupied by another living player as passable; a route that would otherwise detour around that player
   now goes straight through.

## Manual/CLI smoke

1. Run the CLI client with two local/bot players in a narrow corridor map; confirm one can walk through
   the other without rejection or visual snapping/reversion.

## Browser smoke

1. Run the web client with two players in a one-tile-wide corridor; confirm one can move into the tile
   occupied by the other, both render without crashing or glitching while overlapping, and either can
   continue moving away afterward.

## Gates before merge (per constitution)

- `cargo fmt --check`
- `cargo clippy --all-targets -- -D warnings`
- `cargo test --workspace`
