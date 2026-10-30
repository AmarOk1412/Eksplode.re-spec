# Implementation Plan: Bot Behaviours and Difficulty Levels

**Branch**: `012-bot-behaviours-difficulty` | **Date**: 2026-10-30 | **Spec**: [spec.md](spec.md)

**Input**: Feature specification from `specs/012-bot-behaviours-difficulty/spec.md`

## Summary

Give every bot one of three behaviours (Aggressive, Protective, Balanced) picked at random from the
match seed when a match starts, then add three difficulty levels (Easy, Medium, Hard) that slow the
bot's decisions (decision interval and delayed perception of new bombs) and add seeded mistakes, and
finally expose the difficulty as an owner-only room setting (default Medium) in the web and terminal
clients; only the host/owner can change it. In developer mode, the owner can also force the bots'
behaviour (Random by default) from the web developer panel, which gives testers and integration
tests today's exact bot (Balanced + Hard). Today's `DefaultBot` becomes Balanced/Hard unchanged;
behaviours reorder the existing goal
search (hunt, power-up, dig) on top of the shared safety layer; the difficulty travels as an additive
`bot_difficulty` field of `RoomSettings` through the existing `SetRoom` path.

## Technical Context

**Language/Version**: Rust stable, edition 2021 workspace (`core`, `bots`, `net`, `cli`, `web`,
`web-render`); browser ES modules under `crates/web/assets/`.

**Primary Dependencies**: No new crates. Reuses `core::Rng` (SplitMix64), `serde`/`serde_json`,
`ratatui` (terminal UI), existing `find_path`/`DangerMap` in `bots`.

**Storage**: None new. The web client already saves room settings in local storage; the new field is
included.

**Testing**: `cargo test --workspace` (scripted bot scenarios in `crates/bots/tests/behaviour.rs`,
seeded statistics in `crates/bots/tests/soak.rs`, `core` room tests, `net`/`web` room host tests,
`cli` args/setup/online tests); Node tests in `crates/web/js-tests/`.

**Target Platform**: Linux-first server and terminal; existing desktop/mobile browsers.

**Project Type**: Existing Rust workspace: game library, bots library, networked server, terminal and
web clients.

**Performance Goals**: No regression of the 20 Hz tick: Easy/Medium bots compute a fresh decision
only every 6/3 ticks, so per-tick bot cost is equal or lower than today.

**Constraints**: Deterministic from the match seed (Principle IV); bots see only the `Observation`
(Principle V); `DefaultBot::new(seed)` must remain bit-identical (FR-015); old clients without the
field keep working (FR-020); no `unsafe`, no panics on wire data.

**Scale/Scope**: Up to 4 bots per room, 3 behaviours × 3 difficulties; one new room setting.

## Constitution Check

*GATE: checked before research and again after Phase 1 design.*

| Principle/Gate | Pre-design | Post-design |
| --- | --- | --- |
| I. Specification-first | PASS: spec.md written and validated before planning | PASS: every design item traces to FR-001..FR-025 |
| II. Test-first & full coverage | PASS: bot decision logic and settings parsing need failing tests first | PASS: quickstart maps each acceptance scenario to an `f12_usN_asM_*` test; soak tests measure SC-001..SC-004 |
| III. Core/frontend separation | PASS: `core` only gets the `BotDifficulty` data enum | PASS: behaviour/skill logic lives in `bots`; clients only display/edit the setting |
| IV. Deterministic simulation | PASS: behaviour pick and mistakes use seeded `Rng` | PASS: salted RNG for picks, bot RNG for mistakes; no draws at Hard keeps today's streams |
| V. Four-player multiplayer with bots | PASS: same `Bot` trait and `Observation` | PASS: reaction delay only hides information (younger bombs), never adds any; scripted scenarios per behaviour |
| VI. Dual interface parity | PASS: setting in web form/panel and terminal host flag/setup/lobby | PASS: contracts/bots-and-ui.md lists both surfaces; the dev behaviour override follows existing developer mode, which is web-page only for both hosts |
| Minimal deps / no `unsafe` | PASS | PASS |
| No panics / typed errors | PASS: invalid wire values fall back to Medium | PASS: CLI flag errors use `UsageError` |
| Host authority | PASS: owner/lobby checks reuse `SetRoom` validation | PASS: `DevSetBotBehaviour` reuses dev-command owner + dev-mode checks |
| Docs | PASS: README/GAMEPLAY/PROTOCOL/DEVELOPMENT updates planned | PASS: listed in quickstart done criteria |

## Project Structure

### Documentation (this feature)

```text
specs/012-bot-behaviours-difficulty/
├── spec.md
├── checklists/requirements.md
├── plan.md                     # this file
├── research.md                 # Phase 0
├── data-model.md               # Phase 1
├── contracts/
│   ├── room-settings.md        # wire field and CLI flags
│   └── bots-and-ui.md          # bots API, web and terminal controls
├── quickstart.md               # Phase 1
└── tasks.md                    # /speckit-tasks
```

### Source Code (repository root: `game/`)

```text
crates/core/src/
├── room.rs             # BotDifficulty enum; RoomSettings.bot_difficulty, step_bot_difficulty, label
└── lib.rs              # re-export BotDifficulty

crates/bots/src/
├── profile.rs          # new: Behaviour, Skill, Profile
├── default_bot.rs      # with_profile, behaviour priority table, think interval, reaction delay, mistakes
└── lib.rs              # re-exports
crates/bots/tests/
├── behaviour.rs        # scripted scenarios per behaviour/difficulty
└── soak.rs             # SC-001..SC-004 statistics

crates/net/src/
├── protocol.rs         # DevSetBotBehaviour (v2 Request, legacy ClientMsg); DevState.behaviour
├── client.rs           # legacy -> v2 mapping of DevSetBotBehaviour
├── room.rs             # from_wire bot_difficulty; profiles at Start and on takeover; dev bot_behaviour
└── host.rs             # bots_for and legacy takeover use the lobby's difficulty; DevCommand::Behaviour

crates/cli/src/
├── app.rs              # make_bots uses the lobby settings' difficulty (Medium for local games)
├── args.rs             # --bot-difficulty for host only, usage text
├── setup.rs            # Bot difficulty row
└── online.rs           # `d` key for the owner

crates/web/assets/
├── index.html          # selects in create form, lobby panel and dev panel
├── room.js             # defaultRoom / sanitizeRoom / roomReducer / summary
└── app.js              # bind selects, gating for non-owners; devPanel behaviour select + DevState.behaviour

README.md, docs/GAMEPLAY.md, docs/PROTOCOL.md, docs/DEVELOPMENT.md
```

**Structure Decision**: Existing workspace; one new module `crates/bots/src/profile.rs`, all other
changes are additive in existing files.

## Complexity Tracking

*No constitution violations requiring justification.*
