# Implementation Plan: Game Modes and Room Settings

**Branch**: `004-game-modes` | **Date**: 2026-10-23 | **Spec**: [spec.md](spec.md)

**Input**: Feature specification from `/specs/004-game-modes/spec.md`

## Summary

A room becomes a described thing: one self-contained `RoomSettings` (name, mode, board size, round time, friendly fire) that the first person fills in on a **room creation menu**, that every participant sees in the lobby and that only the owner may change before a match. Three modes use it: **Normal** (today's rules plus an **end pattern** that closes the board in the last 60 seconds until one player is left), **Team** (2 against 2 with a friendly-fire switch) and **Chaser** (one chaser with 2 bombs and 3 hearts against three bomb-less runners on a nearly open board with random bonuses). The delivery order follows the request: room menu first, then the end pattern, then the other settings (size, time), then the new modes.

Technical approach (details in [research.md](research.md)):

- Rules stay in `core`: new `room` module (`RoomSettings`, `Mode`, `BoardSize`), new `closing` module (end pattern), `team` and `chaser` rules inside `step`, additive `Config` fields with serde defaults, additive `Participant`, `Flame`, `Outcome`, `Event` and `Observation` fields. Everything is counted in ticks and uses the seeded RNG, so a seed plus settings plus inputs still gives one result (principle IV).
- `net` carries the settings: `Lobby.settings`, `ClientMsg::RoomQuery` answered by `ServerMsg::RoomInfo` (so the page knows, before joining, whether to show the creation menu), `ClientMsg::SetRoom`, an optional `room` in `Join`, new error codes; decoding of the settings is lenient like looks. The host stays the only authority; settings are validated before they are applied.
- `bots` learn the falling blocks, teams and the two Chaser roles.
- `cli` and `web` show and set the settings (terminal setup screen and options; page creation menu and lobby controls) and draw teams, hearts, the chaser mark, announced squares and falling blocks; `web-render` gets the new drawing, the page the menus and sounds.
- One room per server. Nothing in this plan adds room lists, passwords or several rooms; `RoomSettings` is the one value a later network feature will publish.

## Technical Context

**Language/Version**: Rust (stable, 2021 edition) as in the earlier features; presentation JS/HTML/CSS under `crates/web/assets/`.

**Primary Dependencies**: none new (`serde`, `serde_json`, `ratatui`, `tungstenite` stay).

**Storage**: none on the server; the page keeps the player's look and sound settings in `localStorage` as before; it may remember the last room settings (`eksplode.re.room.v1`) as menu defaults.

**Testing**: `cargo test --workspace` (core scenario tests per rule, net decoding and authority tests, bots scenario tests, cli view and key tests, web-render command tests), `node --test` for the page models (room menu, lobby settings, HUD, sounds), the browser smoke script, the pty check, `scripts/check-scenarios.sh` extended with the 004 scenario ids.

**Target Platform**: Linux first; the browser page and the terminal client.

**Project Type**: Cargo workspace with `core`, `bots`, `net`, `cli`, `web`, `web-render` plus the page assets.

**Performance Goals**: every mode and size keeps the 20 Hz tick; key-to-snapshot p95 at most 200 ms on the large board; the wasm module stays at most 300 KB and the page at most 500 KB.

**Constraints**: protocol version 1 and additive changes only; no panics on external input; `core` stays pure; rules only in `core` (the page and the terminal draw and send settings, they never decide); at most 4 participants.

**Scale/Scope**: 3 modes, 3 board sizes, 4 round times, 4 slots; the end pattern covers at most 25 x 19 squares; five delivery stages plus polish.

## Constitution Check

*GATE: passed before Phase 0, re-checked after Phase 1.*

| Principle | Status | How the plan complies |
|-----------|--------|-----------------------|
| I Specification-First | PASS | `spec.md` exists with FR-001..FR-030 and scenarios; every stage traces to them; contracts are written before tasks. |
| II Test-First & coverage | PASS | Every acceptance scenario gets an automated test named `f4_usN_asM_*` (the convention of feature 003); the quickstart lists the commands; rules are tested with hand-built states. |
| III Core/Frontend separation | PASS | Modes, the end pattern, hearts, teams, bonus spawning and win conditions live in `core`; the page and the terminal only show them. The room menu sends a settings value; the host validates and applies it. |
| IV Deterministic simulation | PASS | The pattern order is derived from the arena, the chaser and the bonus positions come from the state's seeded RNG, all timers count ticks; settings are part of the input of a match. |
| V Four-player multiplayer with bots | PASS | Still 4 slots; every role (team member, chaser, runner) can be a human or a bot using the public observation and `PlayerInput` only; bots are tested with scripted scenarios. |
| VI Dual interface parity | PASS | The terminal gets options, a setup screen, the same lobby information and the same drawing of teams, hearts and falling blocks; the page gets the menu. A stage is done only when both work. |
| Technology constraints | PASS | No new dependency; no `unsafe`; settings decoding returns typed errors or defaults; `unwrap`/`expect` only in tests. |
| Quality gates | PASS | fmt, clippy `-D warnings`, `cargo test --workspace`, node tests and docs tests (README, PROTOCOL, WEB, GAMEPLAY) are part of every stage gate. |

Post-design re-check: the data model and contracts keep all rules in `core` and validate every external value; no violation to justify.

## Project Structure

### Documentation (this feature)

```text
specs/004-game-modes/
├── plan.md              # This file
├── research.md          # Phase 0: decisions and alternatives
├── data-model.md        # Phase 1: settings, modes, roles, hearts, end pattern, additions to existing types
├── quickstart.md        # Phase 1: how to run and validate each stage
├── contracts/
│   ├── room-and-protocol.md   # RoomSettings on the wire, RoomInfo, SetRoom, Join.room, errors
│   ├── modes-and-rules.md     # rules of Normal, Team, Chaser, the end pattern, step order, results
│   ├── cli-and-page.md        # options, terminal setup screen, page menu, lobby, HUD, sounds
│   └── render-and-bots.md     # observation additions used by drawing and bots, bot behaviour
├── checklists/
│   └── requirements.md
└── tasks.md             # Phase 2 (/speckit-tasks, not created here)
```

### Source Code (repository root: `game/`)

```text
crates/core/src/
├── room.rs          # new: Mode, BoardSize, RoomSettings, validation, to_config, participants_needed
├── closing.rs       # new: the end pattern (spiral order, announcements, landings)
├── team.rs          # new: team of a slot, friendly-fire test, team result
├── chaser.rs        # new: role choice, hearts and shield, random bonus schedule, chaser arena
├── config.rs        # + mode, friendly_fire, closing and chaser tuning (serde defaults)
├── arena.rs         # + Chaser layout, clear areas for the center
├── state.rs         # + Participant { team, role, hearts, shield_left }, Flame.owner, events, Outcome variants
├── step.rs          # + step order of contracts/modes-and-rules.md
├── lobby.rs         # + Lobby.settings, participants_needed, can_start by mode
└── observation.rs   # + falling squares for drawing and bots
crates/net/src/
├── protocol.rs      # + RoomQuery, RoomInfo, SetRoom, Join.room, error codes (Lobby carries the settings)
├── room.rs          # new: lenient RoomSettings decoding from loose JSON
└── host.rs          # room open/closed, SetRoom, start checks, derived Config per match
crates/bots/src/     # danger map with falling squares, team and role aware targets, runner fleeing
crates/cli/src/      # setup screen, options, lobby rows, drawing of teams, hearts, falling squares
crates/web/src/      # server options for the initial settings
crates/web/assets/   # room creation menu, lobby settings, HUD (teams, hearts, time), sounds
crates/web-render/src/ # team marks, chaser mark, hearts, announced squares, falling blocks
docs/                # GAMEPLAY.md (modes), WEB.md and README (room menu, options), PROTOCOL.md
```

**Structure Decision**: keep the six crates; new rules in their own small `core` modules so each mode can be tested alone, wire and lenient decoding in `net`, drawing decisions in `web-render`, menus and sounds in the page assets.

## Delivery Stages

| Stage | Scope | Spec | Gate |
|-------|-------|------|------|
| S1 | **Room settings and the creation menu**: `RoomSettings` with Normal as the only mode, `RoomQuery`/`RoomInfo`, `SetRoom`, `Join.room`, owner rules, lobby display, page menu, terminal options and setup screen | US1 AS1-AS5, AS8, FR-001..FR-005, FR-008 (Normal), FR-009, FR-010 | core, net, web and cli tests, node tests, smoke screenshot of menu and lobby |
| S2 | **The end pattern** for Normal: closing module, new round end rule, drawing and sounds, bots avoid falling squares | US2, FR-011..FR-015 | core scenario tests, bots tests, render tests, soak, smoke screenshot |
| S3 | **Other settings**: board sizes and round times wired into the config, page and terminal fit, owner changes | US1 AS6, AS7, FR-006, FR-007 | arena fairness tests on all sizes, fit tests, latency test on the large board |
| S4 | **Team mode** with friendly fire | US3, FR-016..FR-019 | core, bots, net, cli and render tests |
| S5 | **Chaser mode**: roles, hearts, chaser arena, random bonuses, results, bot roles | US4, FR-020..FR-027 | core scenario tests, bots soak of 200 rounds, render and cli tests |
| S6 | Polish: docs, fuzz of the new messages, performance notes, scenario script, manual checklist | FR-028..FR-030, SC-001..SC-011 | all gates, quickstart |

## Complexity Tracking

No constitution violation. Two choices carry cost and are justified in the research: `Flame` gains an owner (needed for friendly fire; avoids a second flame representation) and `Outcome` gains team and role results (a client that does not know a mode cannot decode its result, which is acceptable because it cannot play that mode either).
