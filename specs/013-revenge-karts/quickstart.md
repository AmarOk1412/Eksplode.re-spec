# Quickstart: Revenge Karts

Implementation and validation guide for the revenge-kart feature. Automated and runtime checks below passed; the separate five-person SC-007 playtest remains unperformed. Models: [data-model.md](data-model.md). Rules: [contracts/gameplay.md](contracts/gameplay.md). Wire, inputs, and views: [contracts/snapshots-and-controls.md](contracts/snapshots-and-controls.md).

## Implementation Evidence (2026-11-02)

- The implementation request approves proceeding with the existing specification and plan. Existing edits in feature 004 and web `app.js`/`index.html` are preserved.
- Baseline: `cargo test --workspace --quiet` and `node --test` from `game/crates/web/js-tests` passed before production changes.
- Reuse `core/tests/common/mod.rs::{state_with,add_bomb,idle}` for custom arenas and precise ticks, `bots/tests/common/mod.rs` for public observations, and `web-render/tests/common/mod.rs` for renderer fixtures. New cases use `f13_usN_asM_`; filtered suites must report at least one test.
- Setup inspection found two Git repositories, existing Rust/JS ignore rules, and no Docker, ESLint, Prettier, publishing, Terraform, or Helm configuration requiring additional ignore files.
- Foundation red/green evidence: snapshot kart/source round trips and malformed-field checks failed before additive state, then passed; contradictory observation state failed before sanitization, then passed. The old renderer observation fixture remains byte-for-byte unchanged.
- US1 automated checkpoint passed: `cargo test -p eksplode-re-core -p eksplode-re-cli -p eksplode-re-web-render --quiet` and all browser JS tests. Entry, clockwise/counterclockwise travel, illegal directions, inward launch/capacity, Chaser/practice boundaries, immune/non-colliding karts, maximum perimeter dimensions, opposing terminal/device inputs, spatial kart rendering, and fixed gutter dimensions are covered. Final runtime evidence is recorded below.
- The pre-004 on-board event fingerprint now neutralizes inputs from eliminated slots, preserving its original recorded values while separate deterministic replay tests include active karts. The gutter exposed a tiny-icon minimum-size overflow; icons now shrink within their frames and all existing icon tests pass.
- US2/US4/US3 checkpoints passed: all seven throw/stun cases, exact-start-tick cutoff, real room takeover preserving the complete state, live mixed TCP/WebSocket kart travel/throw/earned-return parity, bot navigation and return/profile continuity, all seven revival cases, stale/refreshed flame and chain-source isolation, safe-square ties/capacity reset, and all-size deterministic first/second kart stints.
- Full core and bot packages (including soak), mixed/lifecycle transport tests, terminal input/view tests, renderer command/interpolation tests, and all browser JS tests passed after integration. Most core acceptance fixtures reuse `round_end.rs` rather than duplicating helpers across neighboring rule test files.
- Driver visibility follow-up: `f13_kart_has_a_visible_driver_with_the_players_custom_look` failed before the fix and passed afterward. Karts now draw the player's inward-facing upper body with current colour, head/face accessories and costume, seated behind the vehicle front. The full renderer suite, renderer Clippy, WASM rebuild and browser suite passed. Live desktop (1280x800) and mobile (390x844) screenshots show the driver; the mobile gutter check found 48 light driver pixels and 36 dark detail pixels without horizontal overflow. Updated preview uses port 8089 to preserve the existing server on 8088.

## Final Validation Results

| Check | Observed Result |
|-------|-----------------|
| Rust gates | `cargo fmt --all --check`, strict workspace/all-target Clippy, and `cargo test --workspace --quiet` passed after implementation and the responsive CSS fix. |
| Browser suite | All Node JS/WASM tests passed against the rebuilt asset. |
| WASM build | `bash scripts/build-wasm.sh` passed; rebuilt renderer with the visible customized kart driver is 240,384 bytes, below 300 KB. Legacy observation JSON remains unchanged. |
| Scenario coverage | All 26 feature-013 acceptance prefixes have registered automated tests; `bash scripts/check-scenarios.sh` passes. Unknown selectors are rejected; registration is supplemented by executed package suites. |
| Existing PTY gate | All Enter variants, host/joined ownership and host-served-page checks passed. |
| Additional real PTY smoke | 24 combinations of Normal/Team, small/medium/large, and P1/P2/P3/P4 each passed own-bomb elimination to kart, right binding, exact minimum size, undersized notice, and resize recovery. Solo practice ended without a kart; Chaser lost three hearts and the runners won without a kart. Existing Python PTY helpers were reused without source changes or a new runner. |
| Desktop browser | At 1280x800 a real elimination produced the visible exterior kart and correct HUD. Clockwise traversal visited all 52 track positions and all corners; canvas contained 190,710 nontransparent pixels. |
| Portrait browser | At 390x844 native emulated touch placed a bomb, entered a kart, and drove a complete counterclockwise circuit through all 52 positions. The kart gutter had 240 nontransparent sprite pixels; no horizontal overflow. |
| Landscape browser | At 844x390 the Team match retained identity/team. A screenshot exposed the HUD/arena overlap; shipped CSS now uses separate grid tracks. Rechecked actual asset: HUD bottom 72, canvas top 76, canvas height 310; no overlap/overflow. |
| Portrait layout after fix | HUD bottom 116, canvas top 120, canvas bottom 628.61, joystick top 650.61; no overlap/overflow. Desktop and both mobile orientations were captured with browser screenshots. |
| Actual revenge return | In the live Team room, ordinary touch/keyboard actions drove to the bottom-left opponent lane, eliminated the opponent, and returned P1 at (2,11), still Team A, with one kill, two bombs placed, reset powers and no kart; the victim became a kart. Independent live TCP/WebSocket tests also verify identical return snapshots. |
| Actual cutoff/rematch | Kart present at public snapshot tick 2400, absent at 2401 (first warning); the Normal round finished and returned to the lobby, then started a fresh round without old temporary state. |
| Input latency | Ten actual DOM-keydown samples measured p95 93 ms to a changed snapshot and 105 ms to the next paint, with focus/visibility emulation enabled. A driver-inclusive probe was discarded because it included automation scheduling delay. Standard-mapped gamepad movement, remapped keyboard/device unit tests, and native touch joystick/bomb actions passed. |
| Overlap fixtures | Both renderer tests retain four identities in same-index fixtures; WASM commands stay in bounds at all corners and terminal separation never changes authoritative track indices. These are renderer fixtures, not a claim that four simultaneous karts keep a zero-survivor round alive. |
| Authority/compatibility | Kart/stun/source metadata round trips; malformed source lengths/types are rejected, invalid/future/contradictory added state is inert, maximum supported large-board source-bearing snapshot fits 64 KiB, and no new event/message/input type is emitted. |
| Hook/checklist handling | No extension configuration was present before/after implementation. Reviewer/spec-quality checklist markers were read only and left unchanged. |
| SC-007 / T064 | BLOCKED: five independent human playtest participants were not available. No comprehension result or timing has been fabricated; T064 remains unchecked. |

FR-001 through FR-021 and SC-001 through SC-006 are supported by the acceptance tests, package gates, and runtime checks above; the task list's requirement traceability identifies their owning tasks. SC-007 is the sole outstanding external acceptance criterion. No branch or commit was created, no new dependency/mode/setting/unsafe block was added, and pre-existing feature-004 and web lobby edits were preserved.

## Prerequisites

- Work from `game/`; Rust stable with rustfmt/clippy, Node.js 20+, and `wasm32-unknown-unknown` for the renderer build. Python 3 is needed for the repository's existing PTY gate.
- The specification and plan must be approved before gameplay implementation. Preserve unrelated working-tree changes; no new branch/worktree is required by this guide.
- Use existing hand-built `GameState::custom` test helpers for precise kills, flight lines, safe-return search, and tick boundaries. Browser manual play alone cannot prove these cases.

## Red-First Focused Checks

Add regression cases to existing neighboring tests before changing rules. Use the `f13_usN_asM_` prefix below so a test name identifies its acceptance scenario. These names are proposed; commands with the filter are coverage checks only after the tests exist. Verify the runner reports a nonzero test count; an empty filtered run is not success.

```sh
cargo test -p eksplode-re-core --features serde f13_
cargo test -p eksplode-re-bots f13_
cargo test -p eksplode-re-net f13_
cargo test -p eksplode-re-cli f13_
cargo test -p eksplode-re-web f13_
cargo test -p eksplode-re-web-render f13_
```

Observe each new rule regression fail for its missing behavior, implement the smallest owning slice, and rerun that exact regression before widening. Use existing Rust/Node test harnesses; no new standalone runner or full test implementation belongs in this guide.

## Acceptance Coverage

Each row requires at least one named automated test; parameterize modes, sizes, friendly fire, and controllers where relevant. Manual screenshots supplement, not replace, rule tests.

| Scenario / Proposed Test Prefix | Required Behavior / Preferred Test Home |
|--------------------------------|------------------------------------------|
| `f13_us1_as1_` | Eligible elimination creates a kart, keeps identity/team; core round and team tests. |
| `f13_us1_as2_` | Both circuit directions, release, all corners, wrap; core movement and client input tests. |
| `f13_us1_as3_` | Up/down/punch ignored, opposing directions neutral; core plus JS/CLI input tests. |
| `f13_us1_as4_` | Throw toward square two from all edges/corners; core punch/bomb tests. |
| `f13_us1_as5_` | Chaser deaths never produce karts; existing Chaser tests. |
| `f13_us1_as6_` | No kart damage, item pickup, collision, or survivor counting; core round tests. |
| `f13_us2_as1_` | First candidate is square two, not square one; core punch tests. |
| `f13_us2_as2_` | Every obstacle type and a moving occupant are skipped on arrival; core punch tests. |
| `f13_us2_as3_` | Hit causes no damage and blocks exactly one second of movement; core movement tests. |
| `f13_us2_as4_` | Bomb/punch remain enabled, repeated stun refreshes; core movement/malus tests. |
| `f13_us2_as5_` | Far-edge fallback and fully blocked preflight refusal; core punch tests. |
| `f13_us2_as6_` | Airborne expiry, landing, and chain behavior unchanged; core punch/chain tests. |
| `f13_us2_as7_` | Teammate stun/damage obey friendly fire on/off; core team tests. |
| `f13_us3_as1_` | Matching kart source earns immediate safe return; core round/bomb tests. |
| `f13_us3_as2_` | Reset powers, not identity/team/stats/owned bombs; core powerup/round tests. |
| `f13_us3_as3_` | Multikill gives one return, simultaneous returns use distinct ordered squares; core round tests. |
| `f13_us3_as4_` | Reject teammate/wrong-owner/pre-death sources; include stale-stint flame and chain-source cases; core chain/team tests. |
| `f13_us3_as5_` | No safe square retains one entitlement; first available square used or result ends; core round tests. |
| `f13_us3_as6_` | Repeat death/return uses a new stint; core round tests. |
| `f13_us3_as7_` | Return-before-win/draw with last opponent or team eliminated; core round/team tests. |
| `f13_us4_as1_` | Exact first-warning tick purges before movement, throw, and revival; core closing tests. |
| `f13_us4_as2_` | Post-cutoff deaths/kills cannot create karts/return, old bombs finish; core closing tests. |
| `f13_us4_as3_` | Already returned player stays living at closing; core closing tests. |
| `f13_us4_as4_` | Round end/rematch clears all temporary fields; core plus net/web lifecycle tests. |
| `f13_us4_as5_` | CLI/web agree on identity/team and transitions; view tests, mixed transport, JS models. |
| `f13_us4_as6_` | Bot navigation, throw, earned return, and takeover preserve state; bot behavior/parity plus host tests. |

Additional required cases: all three room board sizes; maximum valid core dimensions without ring overflow; same-position karts; tick-zero elimination; non-default `tick_hz`; source provenance on refreshed flames; stale source after two revivals; current kart capacity despite old bombs; returned ordinary capacity above reset maximum; entry/return ties; no slot-iteration bias; dynamic line closure after a feasible launch; keyboard/controller/touch/cross-device conflicts; bot difficulty cadence/held-path handling; human-kart navigation; malformed new snapshot fields and old defaults.

Repeat prepared situations with identical seed/actions and compare state plus events. Extend existing determinism coverage to include multiple kart stints, return search, closing cutoff, and mode outcomes. Extend the scenario-coverage script to recognize feature-013 prefixes during implementation if it does not already, rather than assuming old feature filters cover these cases.

## Package and Repository Gates

Run the relevant existing package tests without a filter as well, so legacy punch, movement, Chaser, attribution, and closing behavior is exercised:

```sh
cargo test -p eksplode-re-core --features serde
cargo test -p eksplode-re-bots
cargo test -p eksplode-re-net
cargo test -p eksplode-re-cli
cargo test -p eksplode-re-web
cargo test -p eksplode-re-web-render
cargo test -p eksplode-re-web --test mixed
cargo test -p eksplode-re-web --test docs
cargo test -p eksplode-re-bots --test soak -- --nocapture
```

Before merge, from `game/`:

```sh
cargo fmt --check
cargo clippy --workspace --all-targets -- -D warnings
cargo test --workspace
bash scripts/build-wasm.sh
(cd crates/web/js-tests && node --test)
python3 scripts/pty-enter-check.py
bash scripts/check-scenarios.sh
```

Expected: all tests/gates pass; the rebuilt committed renderer asset stays within the repository's existing size limit; no compatibility or legacy-mode regressions. Do not claim a baseline pass from this planning run: gameplay tests have not been executed here.

## End-to-End Validation

1. Build the renderer as above. Start `cargo run -p eksplode-re-web -- --bind 127.0.0.1 --dev` and open `http://127.0.0.1:8080/` (choose unused ports using documented flags if occupied). Join from a second browser and `cargo run -p eksplode-re-cli -- join 127.0.0.1`. Create Normal mode with four participants, including bots; repeat with Team and friendly fire both ways.
2. While at least two opposing sides remain alive, let a human die before closing. Confirm a small identifiable kart appears at the nearest edge in both browser and terminal. Drive an entire lap in each direction and release controls; check all corner throws target inward, not the travel direction. Repeat with keyboard remapping, gamepad, touch, and terminal bindings.
3. Reproduce precision throw/hit/revival cases through the automated fixtures above. In live play, observe a direct hit, loss of movement for one second without disabling bomb/punch, a qualifying opponent kill, and return to a safe square. Check revived powers reset and old bombs retain ownership. A debug fixture can set a near-closing state in tests; no new production developer command is required.
4. At the first closing warning, confirm all karts vanish, their inputs stop affecting play, and a later explosion cannot return them. A previously returned participant remains living. Finish the round and start a rematch; no temporary state persists. Repeat Chaser to confirm unchanged controls and outcomes.
5. For local terminal parity, run `cargo run -p eksplode-re-cli -- --players 1 --bots 3` and repeat kart movement/throw/return. For each size, test the exact minimum terminal dimensions (40x22, 44x24, 56x28), one smaller dimension, and resize during a kart lap. Required-size notice replaces rendering only when genuinely undersized.
6. Use browser automation on desktop 1280x800 and mobile 390x844, including touch orientation changes. Capture screenshots before/after death and at each edge/corner; assert the canvas has non-background pixels in the projected kart region, sprites stay within fitted bounds, repeated frames show expected movement, wrap interpolation stays outside the board, status text does not overlap, and same-index karts remain identifiable. Use the current page flow; the existing `browser-smoke.mjs` script's old `#join` selector is not a reliable validator for this feature.
7. For SC-007, run the five-person comprehension playtest with documented controls, without coaching. At least four participants must drive a corner, throw inward, and correctly describe revival and cutoff within two minutes of their first elimination. Record the observed results during implementation validation, not as a pre-checked planning result.

## Completion Evidence

Implementation is ready only when every acceptance prefix has a nonzero automated test count, deterministic/legacy/compatibility gates pass, updated user docs explain the mechanic, and browser/terminal smoke evidence and the five-person playtest have been recorded. Update README and relevant GAMEPLAY, PROTOCOL, DEVELOPMENT, and WEB documentation for controls, new snapshot fields, validation steps, and larger terminal requirements. Do not add a room flag, new binding, or Chaser behavior.