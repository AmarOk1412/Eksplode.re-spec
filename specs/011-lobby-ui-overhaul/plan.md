# Implementation Plan: Menu, Lobby, Customization and Scoreboard Reorganization

**Branch**: `011-lobby-ui-overhaul` | **Date**: 2026-10-29 | **Spec**: [spec.md](spec.md)

**Input**: Feature specification from `specs/011-lobby-ui-overhaul/spec.md`

## Summary

Reorganize the web client's navigation into Main Menu → (Create / Join+room-list / Options), rebuild the
lobby around per-slot controls (bot auto-fill, per-slot kick/remove-bot/add-bot, a configuration panel,
a leave icon, and a two-button Ready/Start action row), split Options into Customize/Sound/Controls tabs,
simplify character customization (drop the gender "Look" selector, add a new Costume category, move
randomize to a corner icon), remove the red `:focus-visible` outline, and rework the end-of-match
scoreboard to rank by survival order and show only name/kills/bombs placed. Almost all lobby actions
already exist on the v2 room protocol (`CreateRoom`, `JoinRoom`, `ListRooms`, `SetRoom`, `AddBot`,
`RemoveBot`, `RemovePlayer`, `SetReady`, `Start`, `LeaveRoom`); this feature adds the UI chrome around
them plus three small protocol/core extensions: bot-fill-on-vacancy policy, a slot-targeted bot removal
(today's `RemoveBot` only frees "the highest bot slot"), and per-player `kills` / `bombs_placed` /
elimination-order tracking in `core` needed to drive the new scoreboard.

## Technical Context

**Language/Version**: Rust stable, workspace edition 2021, MSRV 1.75 (`crates/core`, `crates/net`,
`crates/web`, `crates/web-render`); browser-side ES modules/HTML/CSS served from `crates/web/assets/`;
`crates/web-render` compiles to WebAssembly per the existing `scripts/build-wasm.sh` pipeline.

**Primary Dependencies**: No new crates. Reuses `serde`/`serde_json` (wire types), the existing v2 room
protocol in `crates/net/src/protocol.rs`, and the existing `eksplode_re_net::look::Look` /
`eksplode_re_web_render::look::Look` cosmetic types. No new browser dependencies; plain DOM/CSS as today.

**Storage**: No persistence changes. Per-match counters (`kills`, `bombs_placed`, elimination order) live
only in in-memory `core::State` for the duration of a match, same lifetime as existing per-player state.

**Testing**: `cargo test --workspace` (unit tests in `core` for the new counters/ordering and for
slot-targeted bot removal; `net` protocol round-trip tests for the new/changed messages); existing
Node-based JS smoke tests under `crates/web` for the reorganized menu/lobby/options/scoreboard DOM; CDP
browser smoke screenshots for the new lobby and scoreboard layouts, consistent with features 006/007.

**Target Platform**: Linux-first server; existing desktop and mobile browsers already supported by
features 002/006/007. No new platform surface.

**Project Type**: Existing Rust workspace; web-served graphical client plus its authoritative server.

**Performance Goals**: No change to the 20 Hz simulation or existing input-latency targets; this feature
is UI reorganization plus O(1) per-tick bookkeeping (incrementing a counter, recording a tick number),
not a hot-path change.

**Constraints**: No gameplay rule changes beyond the bookkeeping needed for FR-017/FR-018/FR-019 (kills,
bombs placed, elimination order are observational counters, not new interactions); owner-only actions
(kick, remove bot, configure room, Start) remain server-validated regardless of client UI state (FR-016);
no plaintext password exposure beyond what the existing v2 `SetRoom`/password handling already allows;
`unsafe` code remains forbidden; dual-interface parity (Principle VI) — the CLI/terminal client has no
lobby seat chrome or scoreboard table to redesign today, so this feature's UI changes are web-only, but
the new `core` counters (kills, bombs placed) and the slot-targeted bot removal are added in `core`/`net`
so they remain available to the CLI client if it later needs them (no CLI UI work is in scope here).

**Scale/Scope**: Four player slots per room (unchanged `MAX_PLAYERS`); one configuration panel, one
room-list screen, one tabbed options screen, one customize panel, one scoreboard table — no new screens
beyond what six user stories in the spec describe.

## Constitution Check

*GATE: checked before research and again after Phase 1 design.*

| Principle/Gate | Pre-design | Post-design |
| --- | --- | --- |
| I. Specification-first | PASS: spec.md (010/011 split, see spec) covers all six stories before any code | PASS: plan/data-model/contracts trace to FR-001..019 |
| II. Test-first & full coverage | PASS: new counters/ordering/slot-removal need failing tests first | PASS: test plan in quickstart.md covers every acceptance scenario; `cargo fmt --check`/`clippy -D warnings`/`cargo test --workspace` still required before merge |
| III. Core/frontend separation | PASS: kills/bombs-placed/elimination-order live in `core`, not in JS | PASS: data-model.md keeps these as plain `core::State` fields; `web`/`net` only relay them |
| IV. Deterministic simulation | PASS: new counters are derived deterministically from existing tick-ordered events | PASS: no new randomness; bot-fill-on-vacancy and randomize-look use the existing seeded RNG already used for bot/AI and look defaults |
| V. Four-player multiplayer with bots | PASS: bot-fill-on-vacancy and per-slot removal operate within the existing 4-slot `Lobby` | PASS: `Lobby::add_bot`/new `remove_bot_at`/`clear` keep the same slot model; bots remain indistinguishable to game rules |
| VI. Dual interface parity | PASS (with scoped exception, see below) | PASS: new `core`/`net` data is interface-agnostic; only the web client's screens change, matching this feature's scope |
| Technology/minimal deps | PASS: no new dependencies | PASS: unchanged |
| Library must not panic / `Result` errors | PASS: new `Lobby` methods return `LobbyError` like existing ones | PASS: `remove_bot_at`/kick validation return typed errors, no `unwrap`/`expect` outside tests |
| Authority: host validates all client input | PASS: kick/remove-bot/configure/Start already require owner on the host | PASS: FR-016 requires explicit host-side re-validation of ownership for every owner-only v2 request, re-checked in contracts/protocol-changes.md |
| Documentation/merge gates | PASS: README/WEB docs will be updated alongside implementation | PASS: quickstart.md lists doc updates as part of Done criteria |

**Scoped exception note (Principle VI)**: the CLI/terminal frontend has no lobby-seat UI or post-match
scoreboard table to redesign (it already reports results as plain text), so there is no CLI-side screen
to bring to parity for *this* feature's menu/lobby/scoreboard chrome. The underlying data this feature
adds (kills, bombs placed, elimination order) is added in `core`, not web-only, so a future CLI
enhancement could surface it without re-deriving it. This is a scope note, not a constitutional
violation — no gameplay feature is web-only.

## Project Structure

### Documentation (this feature)

```text
specs/011-lobby-ui-overhaul/
├── spec.md
├── checklists/requirements.md
├── plan.md                        # this file
├── research.md                    # Phase 0 output
├── data-model.md                  # Phase 1 output
├── contracts/
│   ├── protocol-changes.md        # new/changed v2 client/server messages
│   └── ui-screens.md              # screen inventory: menu, lobby, options, scoreboard
└── quickstart.md                  # Phase 1 output
```

`tasks.md` is deferred to `/speckit-tasks`.

### Source Code (repository root: `game/`)

```text
crates/core/src/
├── lobby.rs            # add remove_bot_at(slot); bot-fill-on-vacancy helper used by net/web host logic
├── state.rs            # Player: add kills: u32, bombs_placed: u32, eliminated_order: Option<u32>
├── step.rs / closing.rs / bomb.rs
│                       # increment bombs_placed on placement; attribute kills and record
│                       # eliminated_order at the existing elimination sites (step.rs:185, closing.rs:125)

crates/net/src/
├── protocol.rs         # v2 Request: RemoveBot gains an explicit slot; ServerMsg/Snapshot carries
│                       # kills/bombs_placed/eliminated_order per slot for the results screen
├── look.rs             # Look: drop `gender` from the wire-level customization surface (SetLook),
│                       # add `costume`; old `gender` field tolerated-but-ignored for backward reads
├── room.rs / host.rs   # bot-fill-on-vacancy policy call sites (room creation, slot vacated pre-match)

crates/web-render/src/
├── look.rs             # Costume variant list; Gender becomes an internal rendering-only detail
│                       # (no longer settable from the wire), default assigned the way it is today
├── iso3d.rs            # costume sprite rendering alongside existing head/face overlays

crates/web/assets/
├── index.html          # main menu, room-list (join), tabbed options, lobby header
│                       # (config icon, leave icon, per-slot controls), scoreboard table head
├── app.js              # menu navigation/state machine, lobby slot rendering, scoreboard rows/order
├── room.js             # roomControls() gating extended for the configuration panel
├── look.js             # editor reducer: drop gender, add costume, randomize-icon wiring
└── style.css           # focus-visible color fix; new icon/panel styles
```

**Structure Decision**: Existing five-crate workspace (`core`, `net`, `web`, `web-render`, `cli`) plus
`bots`; no new crates. All changes are additive within existing modules — no restructuring of the
workspace layout.

## Complexity Tracking

*No constitution violations requiring justification.*
