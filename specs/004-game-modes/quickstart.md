# Quickstart: Game Modes and Room Settings

Run from `game/`. Stage gates are cumulative: after every stage run the common gate.

## Common gate

```bash
cargo fmt --all -- --check
cargo clippy --workspace --all-targets -- -D warnings
cargo test --workspace
(cd crates/web/js-tests && node --test)
scripts/check-scenarios.sh      # includes the f4_* ids from this feature
```

## Try it

```bash
# Browser: start the server, open the page, the creation menu appears first
cargo run -p eksplode-re-web -- --bind 127.0.0.1:8080

# Server with defaults for the menu
cargo run -p eksplode-re-web -- --mode team --size medium --time 5 --friendly-fire off

# Terminal host with the setup screen, or with options
cargo run -p eksplode-re-cli -- host
cargo run -p eksplode-re-cli -- host --mode chaser --size large --time 3 --no-setup

# Local game in the terminal with a mode
cargo run -p eksplode-re-cli -- --mode normal --time 2
```

Use `DevPauseBots` and the dev panel (`--dev`) to freeze bots while checking the end pattern; set a 2-minute round to see the pattern after 60 seconds.

## Stage checks

| Stage | Command | What to see |
|-------|---------|-------------|
| S1 | `cargo test -p eksplode-re-net f4_us1` and `node --test` | creation menu, lobby settings, owner-only change |
| S2 | `cargo test -p eksplode-re-core f4_us2` | announce, landing, crush, round always ends |
| S3 | `cargo test -p eksplode-re-core f4_us1_as6 f4_us1_as7` and the large-board latency test | sizes fair, times applied |
| S4 | `cargo test -p eksplode-re-core f4_us3` | teams, friendly fire on and off |
| S5 | `cargo test -p eksplode-re-core f4_us4` and `cargo test -p eksplode-re-bots soak_chaser` | roles, hearts, bonuses, results |
| S6 | docs tests, fuzz tests, smoke script, pty check | docs complete, no panics, screenshots per mode |

## Manual acceptance (browser, two tabs)

1. Open the page: the creation menu shows; create a Team room, medium, 5 minutes, friendly fire off.
2. Open a second tab: the join screen shows the room summary; join; see teams A and B.
3. As owner change the mode to Chaser; the second tab updates; start; chaser at the center with 3 hearts, runners without bombs, bonuses appearing.
4. Rematch in Normal with a 2 minute round; after 60 seconds red squares announce and blocks fall until one player is left.
5. Repeat on a phone-sized window and with the terminal client joined to the same server.

## Scenario ids

`f4_usN_asM_<name>` (N = user story 1..4, M = acceptance scenario); `scripts/check-scenarios.sh` verifies that every acceptance scenario of `spec.md` has a test with its id.
