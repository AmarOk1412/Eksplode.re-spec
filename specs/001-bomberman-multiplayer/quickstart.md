# Quickstart & Validation Guide

> **Updated for the real behaviour** (feature 002, task T084). Stage 4 (native isometric GUI) was
> **replaced by the web page** described in `specs/002-web-client/quickstart.md`. Package names
> are `eksplode-re-core`, `eksplode-re-bots`, `eksplode-re-net`, `eksplode-re-cli`, `eksplode-re-web` and
> `eksplode-re-web-render`.

All commands are run from `game/` (its own git repository).

## Prerequisites
- Rust (`rustc --version`), a terminal >= 80x24.

## Stage 0 - Bootstrap
```sh
cd game && git config core.hooksPath .githooks
cargo fmt --check && cargo clippy --workspace --all-targets -- -D warnings && cargo test --workspace
```
Expected: all pass. A commit with a subject that is not `part: desc` (<= 80 chars) is rejected.

## Stage 1 - Core + CLI (US1)
```sh
cargo test -p eksplode-re-core
cargo run -p eksplode-re-cli -- --players 1 --bots 0 --seed 42
```
Expected: arena shows; arrows move; Space places a bomb that explodes after 3 s, destroys
blocks, chains, eliminates players; round result shown.

## Stage 2 - Bots (US2)
```sh
cargo test -p eksplode-re-bots
cargo run -p eksplode-re-cli -- --players 1 --bots 3 --seed 42
cargo test -p eksplode-re-bots --test soak -- --nocapture   # 100 four-bot matches (SC-003)
```
Expected: bots flee blasts and do not trap themselves; soak test reports 100/100 finished.

## Stage 3 - Multiplayer (US3)
```sh
cargo test -p eksplode-re-net
cargo run -p eksplode-re-cli -- host --bots 1          # terminal A
cargo run -p eksplode-re-cli -- join 127.0.0.1:7878    # terminal B (and C)
```
Expected: lobby shows joined players; the host starts with Enter (a joiner gets "Only <name> can
start the match"); all see the same state and result; killing a client terminal lets the match
continue with a bot; a 5th join is refused.

## Stage 4 - Graphical version (replaced by the web page)
The native window was dropped. See `specs/002-web-client/quickstart.md`:
```sh
cargo run -p eksplode-re-web -- --bots 1       # open http://127.0.0.1:8080/
cargo test -p eksplode-re-web-render           # isometric view, cubes, draw commands
```
Expected: isometric arena, four coloured cubes, same rules and outcomes as the terminal for the
same seed and inputs.

## Acceptance scenario -> test map (maintained in tasks)
Each spec acceptance scenario `USx-ASy` has a test named `usX_asY_*` in the crate that implements
it (`core`: US1; `bots`: US2; `net`: US3). `cargo test --workspace` must show every id. For
feature 002 the script `scripts/check-scenarios.sh` greps for missing ids.

## Definition of done per stage
fmt + clippy + tests green, README updated, commits follow `part: short desc`, spec scenarios for
the stage covered.
