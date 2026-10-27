# Implementation Plan: Bomberman Multiplayer Game

**Branch**: `001-bomberman-multiplayer` | **Date**: 2026-10-22 | **Spec**: [spec.md](spec.md)

**Input**: Feature specification from `/specs/001-bomberman-multiplayer/spec.md`

## Summary

A 4-participant Bomberman delivered in four incremental stages, each a usable product:
(1) terminal game with local humans, (2) local bots, (3) host/join network multiplayer,
(4) isometric graphical client with cubic players. All rules live in a pure, deterministic `core`
crate; bots, networking, CLI and GUI are separate crates depending on it. The host runs the
authoritative simulation; clients send inputs and render the snapshots/events they receive.

The generated code lives in its own git repository in the subdirectory `game/` of this
workspace. The Spec Kit artifacts (`.specify/`, `specs/`) stay in the outer workspace.
Commits follow the project style (see "Commit Convention").

## Technical Context

**Language/Version**: Rust stable, edition 2021 (toolchain here: 1.92), `#![forbid(unsafe_code)]` in all crates

**Primary Dependencies**: `serde` + `serde_json` (protocol), `ratatui` + `crossterm` (CLI), `macroquad` (GUI); `core`, `bots` and the simulation use std only; own seeded SplitMix64 RNG (no `rand`)

**Storage**: N/A (no persistence; replay = seed + input log, used in tests only)

**Testing**: `cargo test --workspace` (unit tests per module, integration tests per crate, loopback network tests, ratatui `TestBackend` for CLI rendering, pure projection/depth-order tests for GUI); `cargo clippy --all-targets -- -D warnings`; `cargo fmt --check`

**Target Platform**: Linux first (terminal >= 80x24 for CLI; X11/Wayland + OpenGL for GUI)

**Project Type**: Cargo workspace: library crates + two binaries

**Performance Goals**: simulation tick 20 Hz; GUI 60 fps rendering with interpolation; < 200 ms action-to-visible latency for 95% of actions on LAN

**Constraints**: deterministic `step(state, inputs)`; no wall-clock/global state in `core`; typed errors, no panics on external input; host authoritative

**Scale/Scope**: 4 participants, one 15x13 arena, 5 crates, a few thousand lines

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principle | Status | How the plan satisfies it |
|-----------|--------|---------------------------|
| I. Specification-First | PASS | Spec approved before this plan; each task will cite FR/US ids |
| II. Test-First & Coverage | PASS | Red-Green per task; acceptance-scenario-to-test map in `quickstart.md`; fmt/clippy/test gates in every stage |
| III. Core/Frontend Separation | PASS | `core` has zero I/O deps; `cli`/`gui` contain no rules |
| IV. Deterministic Simulation | PASS | Tick-based `step`, injected seeded RNG, tick counter as time |
| V. Four-Player with Bots | PASS | One `PlayerInput` interface for human/remote/bot; bots read `Observation` only |
| VI. Dual Interface Parity | PASS | CLI and GUI share `core` + `net` client; parity test replays the same input log through both view models |
| Tech constraints | PASS | Workspace crates `core`, `bots`, `net`, `cli`, `gui`; typed errors; host-authoritative; minimal deps justified in research.md |

Post-design re-check (after Phase 1): PASS, no violations; Complexity Tracking not needed.

## Project Structure

### Documentation (this feature)

```text
specs/001-bomberman-multiplayer/
├── plan.md
├── research.md
├── data-model.md
├── quickstart.md
├── contracts/
│   ├── network-protocol.md
│   └── cli-interface.md
└── tasks.md             # created by /speckit-tasks
```

### Source Code (separate git repository: `game/`)

```text
game/                         # its own git repo (git init here)
├── Cargo.toml                # workspace
├── README.md                 # controls, host/join, bots (FR-018)
├── rust-toolchain.toml
├── .githooks/commit-msg      # enforces commit subject style
├── crates/
│   ├── core/                 # pure rules, no I/O
│   │   ├── src/{lib,arena,state,action,step,bomb,powerup,rng,error,observation}.rs
│   │   └── tests/            # scenario tests mapped to spec acceptance scenarios
│   ├── bots/                 # Bot trait + default bot (danger map, BFS)
│   │   ├── src/{lib,danger,pathing,default_bot}.rs
│   │   └── tests/
│   ├── net/                  # protocol, host, client (std TCP + threads)
│   │   ├── src/{lib,protocol,codec,host,client,lobby}.rs
│   │   └── tests/            # loopback integration tests
│   ├── cli/                  # binary `eksplode-re-cli` (ratatui)
│   │   ├── src/{main,app,input,view,lobby}.rs
│   │   └── tests/
│   └── gui/                  # binary `eksplode_re-gui` (macroquad)
│       ├── src/{main,app,iso,cube,input}.rs
│       └── tests/            # iso projection + depth sort (pure)
└── .github/workflows/ci.yml  # fmt, clippy, test
```

**Structure Decision**: multi-crate Cargo workspace in a dedicated `game/` repository, matching the
constitution's `core`/`bots`/`cli`/`gui` layout plus a `net` crate. Crates are added stage by
stage, so each stage's commit history is self-contained.

## Delivery Stages (spec FR-020)

Each stage ends with all gates green, README updated, and a commit series.

| Stage | Scope | Spec stories | New crates |
|-------|-------|--------------|------------|
| 0 | Bootstrap: `git init` in `game/`, workspace, CI gates, commit hook, README skeleton | - | workspace |
| 1 | Core rules + basic CLI, 1-4 local humans (shared keyboard) | US1 | `core`, `cli` |
| 2 | Local bots, lobby slot selection, bot-vs-bot soak test | US2 | `bots` |
| 3 | Host/join multiplayer, remote lobby, disconnect handling | US3 | `net` |
| 4 | Isometric GUI with cubic players, same lobby/modes as CLI | US4 | `gui` |

## Key Design Decisions (details in research.md and data-model.md)

- **Tick simulation**: `step(&mut GameState, &[PlayerInput; 4]) -> Vec<Event>` at 20 Hz. Fuse 60 ticks (3 s), flames last 10 ticks, round limit 3600 ticks (3 min) then draw.
- **Movement**: tile-based with a per-player cooldown (base 4 ticks per move, power-up reduces it); conflicts resolved by a rotating player order `(tick + index) % 4`.
- **Bots**: `trait Bot { fn decide(&mut self, obs: &Observation) -> PlayerInput }`; `Observation` is the public state view humans have, so bots cannot cheat.
- **Networking**: host is authoritative; clients send `PlayerInput` per tick; host broadcasts a `Snapshot` each tick plus events. Newline-delimited JSON over TCP, threads and `mpsc` channels. A disconnected slot is taken over by a bot.
- **GUI**: macroquad 2D; isometric projection `sx = (x - y) * W/2`, `sy = (x + y) * H/2 - z`; painter's algorithm sorted by `(x + y, z)`; cubes drawn as three shaded faces in the player's colour.
- **Shared client logic**: lobby state machine and input mapping live in testable modules; CLI and GUI only render.

## Commit Convention

Every commit in `game/` uses:

```text
part: short desc (max 80 chars for the first line)

long description to explain the feature
on multiple lines
```

- `part` is the area: `core`, `bots`, `net`, `cli`, `gui`, `build`, `docs`, `test`.
- First line <= 80 characters, imperative mood. The body wraps at about 72 chars and says why and what.
- One logical change per commit; tests are committed with (or before) the code they cover.
- A `commit-msg` hook in `game/.githooks/` enforces the subject format and length.

## Complexity Tracking

No constitution violations; section intentionally empty.
