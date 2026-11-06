# Implementation Plan: Revenge Karts

**Branch**: `main` (unchanged) | **Feature ID**: `013-revenge-karts` | **Date**: 2026-11-02 | **Spec**: [spec.md](spec.md)

**Input**: Feature specification from `specs/013-revenge-karts/spec.md`. The setup script reports the feature ID as `BRANCH`; `git branch --show-current` reports `main`. No branch or game worktree is created by this planning workflow.

## Summary

Give eliminated Normal and Team participants an outside-board kart until closing starts. Implement perimeter movement, inward throws, direct-hit immobilization, stint-aware revenge credit, and deterministic returns entirely in the pure core. Reuse existing input messages and punched-bomb travel without changing Chaser rules. Expose additive snapshot fields to bots and both renderers while keeping `alive` reserved for on-board survivors.

The main integration risks are revival from stale bomb/flame ownership, off-by-one immobilization, a throw on the closing-start tick, and incorrectly ending a round before returns are resolved. Phase 0 decisions are in [research.md](research.md); Phase 1 defines [data-model.md](data-model.md), [gameplay contracts](contracts/gameplay.md), [snapshot and control contracts](contracts/snapshots-and-controls.md), and [quickstart.md](quickstart.md). This command ends at design, without gameplay implementation or task generation.

## Technical Context

**Language/Version**: Rust stable, edition 2021, workspace minimum Rust 1.75; existing browser JavaScript remains presentation/input glue.

**Primary Dependencies**: Existing workspace crates `eksplode-re-core`, `eksplode-re-bots`, `eksplode-re-net`, `eksplode-re-cli`, `eksplode-re-web`, and `eksplode-re-web-render`; existing serde/serde_json, ratatui/crossterm and networking runtime. No new dependency or game engine is needed for this extension of the existing engine.

**Storage**: Round-local in-memory state only; no database, persistence, room setting, or asset download.

**Testing**: Rust unit/integration tests, serde compatibility fixtures, Node's existing JS test runner, terminal rendering/PTY checks, browser screenshots and canvas-pixel assertions. Use failing behavior tests before implementation; names and coverage groups are defined in the quickstart.

**Target Platform**: Linux terminal and authoritative server; desktop/mobile browsers with the existing WebAssembly renderer and keyboard, controller, and touch bindings.

**Project Type**: Existing multiplayer game workspace with pure rules, bot AI, shared TCP/WebSocket observations, terminal UI, and WASM graphics.

**Performance Goals**: Preserve the default 20 Hz simulation and existing 200 ms p95 input-to-visible-update expectation. All three room board sizes must render a complete perimeter without clipping. A revival search is a deterministic scan of the board for at most four participants; add no unbounded history or per-frame network round trips.

**Constraints**: Core remains I/O-free and deterministic; no new unsafe code; host-authoritative actions; snapshots remain below the existing 64 KiB limit in supported-room scenarios. One second uses `config.tick_hz`, not a hardcoded 20. Geometry uses wider integer arithmetic than `u8` before validation/casting. Existing non-kart simulation ordering is preserved except for explicitly specified return resolution.

**Scale/Scope**: Up to four humans/bots, Normal and Team only; small 15 x 13, medium 19 x 15, large 25 x 19. Core configuration also accepts larger odd dimensions, so perimeter indices use `u16`, not `u8`. Existing unrelated changes in `game/` and feature 004 are preserved.

## Constitution Check

*Pre-research evaluation and post-design recheck against constitution 1.4.0.*

| Gate | Before Research | After Design | Evidence / Required Implementation Gate |
|------|-----------------|--------------|-----------------------------------------|
| I. Specification first | PASS | PASS | Feature 013 exists with a complete quality checklist. User requested planning; no gameplay code is written here. Implementation follows approval of the design. |
| II. Test first and rule coverage | PASS | PASS | Quickstart maps every story acceptance scenario to named red-first test groups; protocol, bot, and cross-interface tests are mandatory. Future implementation must demonstrate red then green. |
| III. Core/frontend separation | PASS | PASS | Core decides track geometry, targets, hits, eligibility, and return positions. Clients only normalize physical inputs and display authoritative observations. |
| IV. Deterministic simulation | PASS | PASS | Tick timers, round-local stint IDs, fixed slot/coordinate ties, no wall clock, and no new randomness. |
| V. Four players with bot parity | PASS | PASS | Same `PlayerInput` and `Observation`; kart decisions precede the bot's current dead-player return, preserving existing difficulty cadence. |
| VI. Dual interface parity | PASS | PASS | Local/online CLI and web require kart controls, outside-board rendering, stun and return display, and mixed-transport validation. |
| Platform, dependency, and security constraints | PASS | PASS | No new crates or unsafe operations. Add serde defaults and validate new bounded state; do not introduce new event enum variants that older clients cannot decode. |
| Merge quality gates | PASS | PASS | Plan requires fmt, clippy, workspace tests, JS tests, WASM build, docs checks, and frontend smoke checks. They are implementation gates, not claimed executed by this planning run. |

No unjustified violation or unresolved design question remains. The five-person comprehension playtest in SC-007 supplements, and does not replace, automated acceptance coverage.

## Project Structure

### Documentation (this feature)

```text
specs/013-revenge-karts/
  spec.md
  plan.md
  research.md
  data-model.md
  quickstart.md
  checklists/requirements.md
  contracts/gameplay.md
  contracts/snapshots-and-controls.md
```

`tasks.md` is a later `/speckit-tasks` output and is deliberately not generated here.

### Source Code (repository root)

```text
game/
  crates/core/src/
    state.rs, step.rs, bomb.rs, punch.rs, closing.rs, observation.rs, lib.rs
    kart.rs                         # proposed focused rules module
  crates/core/tests/                 # extend movement, punch, chain, closing, round_end, team, determinism
  crates/bots/src/default_bot.rs     # kart dispatch and public-state decisions
  crates/bots/tests/                 # extend behavior/difficulty/soak coverage
  crates/net/src/                    # unchanged input envelope; snapshot compatibility
  crates/net/tests/                  # extend serde and authority coverage
  crates/cli/src/                    # local/online input and perimeter/status rendering
  crates/cli/tests/                  # extend input, view, online, resize tests
  crates/web/assets/                 # input reduction, participant status, human-kart navigation
  crates/web/js-tests/               # extend keyboard/controller/touch/status tests
  crates/web/tests/                  # extend mixed transport, lifecycle, docs tests
  crates/web-render/src/             # view bounds, kart sprite, interpolation, stun display
  crates/web-render/tests/           # extend commands/view/input snapshot tests
  scripts/check-scenarios.sh         # include feature-013 acceptance names if not recognized
  docs/                             # GAMEPLAY, PROTOCOL, DEVELOPMENT, WEB as needed
  README.md
```

**Structure Decision**: Extend existing owners and tests rather than add a new subsystem or crate. `kart.rs` is the only proposed new rules file: it isolates geometry and lifecycle helpers while `step.rs` owns ordering, `bomb.rs` owns provenance propagation, and `punch.rs` owns shared flight/landing. Prefer existing test files; create a dedicated kart test module only if those become unwieldy.

## Phase 0: Research Outcome

Resolved: existing owner-bitmask credit, actual chain semantics, flame merging, perimeter coordinates, initial second-square flight, dynamic landing failure, precise movement stun, cutoff precedence, winner ordering, serde compatibility, conflicting-input reduction, bot gating, and renderer bounds. Decisions and rejected alternatives are recorded in [research.md](research.md).

## Phase 1: Design Outcome

1. Add optional participant kart state and an independent immobilization timer, plus optional bomb and flame revenge provenance. Keep cumulative match statistics and actual active-bomb counts intact through death and return.
2. Introduce a deterministic virtual perimeter index; never mutate arena tiles for karts. The core exposes projection helpers shared with bot targeting and renderers.
3. Gate karts at tick entry when closing is due. Preserve normal moves, flight, slides, punches, detonation, and falling-block order. Apply flight hits, gather all eliminations and eligible revenge sources, create karts for newly eliminated survivors, resolve returns in slot order, then evaluate the existing result. Do not let a newly created kart act until the next tick.
4. Keep direct-hit stun independent of maluses, lasting exactly `tick_hz` subsequent movement opportunities. Reuse `Flight`; provenance identifies kart throws while an initial launch at square one makes the first hop test square two.
5. Add defaulted snapshot fields without new event variants or input/message types. Derive kart/stun/return effects from observation changes; continue emitting existing compatible bomb and elimination events.
6. Draw a real perimeter position in both frontends. Reserve one tile of visual gutter around the arena for the full round; update terminal minimum-size and WASM fitting tests. Do not rely on the current fixed 16-pixel canvas margin to fit a vehicle at arbitrary tile scales.
7. Map every acceptance scenario and edge case to deterministic tests, then run shared-state and rendering validation. The quickstart distinguishes proposed regression names from existing runnable package commands.

## Risks and Verification

| Risk | Required Discriminating Check |
|------|-------------------------------|
| Old throws revive a later kart stint | Kill with a previous-stint bomb or lingering flame; no return despite unchanged normal kill metrics. |
| Chain-trigger owner incorrectly steals credit | Kart bomb triggers another owner's bomb; only actual blast contributors to the victim square can qualify. |
| Revival accidentally delays/avoids an existing result | Simultaneous last kill with safe return versus no safe return: evaluate alive-only victory after eligible returns. |
| Closing checked too late | On exact start tick request a throw and produce a qualifying kill: zero new throws and returns, old bombs still resolve. |
| Stun one tick too short | Hit after movement; block exactly the next `tick_hz` movement opportunities, then allow movement. Repeat at a non-default tick rate. |
| Legacy client snapshot failure | Old-shape fixture decodes to neutral state; new snapshot stripped of additive fields is legacy-decodable, with no unknown event variants. |
| Input conflict lost before reaching core | Adapter tests deliver simultaneous left/right to neutral `mv`, while ordinary alive-player input precedence stays unchanged. |
| Human kart incorrectly treated as an abandoned match | Browser navigation retains a participating human kart when every human has `alive = false`. |
| Karts clipped or layout shifts mid-round | Fixed gutter bounds at desktop/mobile sizes, corner/wrap interpolation, terminal minimum/exact/undersized dimensions. |

## Complexity Tracking

No constitution exceptions. Per-stint provenance is required because `Flame.owners` alone cannot distinguish old on-board bombs or previous kart lives; it is bounded per owner rather than an unbounded kill history. No additional persistence, capability negotiation, dependencies, or room options are introduced.
