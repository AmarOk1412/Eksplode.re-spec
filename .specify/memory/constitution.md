<!--
Sync Impact Report
- Version change: 1.3.0 → 1.4.0
- Modified principles: none
- Modified sections: Development Workflow & Quality Gates (allow a per-feature `game_spec_<id>`
  git worktree for parallel feature work, merged into `game`'s main branch before completion)
- Added sections: none
- Removed sections: none
- Follow-up TODOs: none
-->
# Eksplode.re Constitution

## Core Principles

### I. Specification-First (NON-NEGOTIABLE)
Every feature MUST have an approved specification (`/speckit-specify`) and plan before code is
written. Code MUST trace back to a requirement in a spec; behavior not in a spec MUST be added to
the spec first. Rationale: the game has many interacting rules (bombs, chain reactions, power-ups,
win conditions) that are only correct if they are written down and testable.

### II. Test-First & Full Coverage of Game Logic (NON-NEGOTIABLE)
Tests MUST be written before implementation and observed failing (Red-Green-Refactor). All game
rules, bot decision logic, and protocol/serialization code MUST have unit tests. Every spec
acceptance scenario MUST map to at least one automated test. `cargo test` MUST pass with zero
failures, and `cargo clippy -- -D warnings` and `cargo fmt --check` MUST pass before merge.
Rationale: the project is required to be tested and specified.

### III. Core/Frontend Separation
Game rules live in a pure, I/O-free `core` library crate (no rendering, no networking, no
wall-clock or global state). The CLI and graphical frontends MUST be thin clients that depend on
`core`, never the reverse. Frontends MUST NOT contain game rules. Rationale: a single rule
implementation guarantees the CLI and GUI behave identically and keeps logic unit-testable.

### IV. Deterministic Simulation
The game advances through discrete ticks: `state' = step(state, inputs)`. Given the same initial
state, seed, and ordered inputs the result MUST be identical. Randomness MUST come from an
injected seeded RNG; time MUST be an input (tick count), never read directly. Rationale: enables
reproducible tests, replays, bot evaluation, and consistent multiplayer state.

### V. Four-Player Multiplayer with Bots
A match MUST support up to 4 players. Each slot MUST be fillable by a human (local or
networked) or a bot, using one common input interface (`PlayerInput` per tick). Bots MUST use
only the same public game state and actions available to humans (no cheating) and MUST be
unit-tested with scripted scenarios (e.g., escaping a blast radius, not self-trapping,
targeting destructible blocks). Rationale: parity between humans and bots keeps the game fair
and bots testable.

### VI. Dual Interface Parity
The project MUST ship both a CLI (terminal) version and a graphical version of the game; the
graphical version is a web page served by the game server. Both
MUST support the full ruleset, 4 players, and bots. Any new gameplay feature MUST be available
in both before it is considered done. Frontend code MUST be tested where feasible (input
mapping, state-to-view models); pure rendering is exempt but MUST be smoke-run. Rationale: both
interfaces are required deliverables.

## Technology & Platform Constraints

- Language: Rust (stable toolchain, 2021 edition or newer), organized as a Cargo workspace with
  crates for `core`, `bots`, `net`, `cli`, `web` (server) and `web-render` (WebAssembly
  graphics). Networking lives outside `core`. The graphical version is a web page: its
  rendering runs as WebAssembly built from `web-render`, and presentation-only HTML/JS lives
  under `crates/web/assets/`; browser code MUST NOT contain or decide game rules.
- `unsafe` code is forbidden unless justified in the plan and isolated with its own tests. The
  only allowed exception is the WebAssembly memory exchange in
  `crates/web-render/src/ffi.rs`.
- Dependencies MUST be minimal and justified in the plan; `core` MUST NOT depend on rendering or
  terminal libraries.
- Library code MUST NOT panic on invalid input or network data; it MUST return typed errors
  (`Result`). `unwrap`/`expect` are allowed only in tests and provably infallible cases.
- Multiplayer networking MUST treat the host/server state as authoritative and validate all
  client inputs.
- Target platform: Linux first; other platforms MUST NOT be broken knowingly.

## Development Workflow & Quality Gates

- Work flows through Spec Kit: constitution, specify, plan, tasks, implement, converge.
- Each task MUST land with its tests; a task is done only when tests pass and the relevant spec
  acceptance scenarios are covered.
- Before merge: `cargo fmt --check`, `cargo clippy --all-targets -- -D warnings`, and
  `cargo test --workspace` MUST pass.
- Bug fixes MUST start with a failing regression test.
- Public APIs in `core` MUST have doc comments; user-facing controls and CLI flags MUST be
  documented in the README.
- Changes that deviate from a spec or plan MUST update the spec or plan in the same change.
- Each commit MUST contain one coherent feature, fix, or maintenance concern and remain small
  enough for a human to review. Unrelated changes MUST be split into separate commits. A change
  that cannot reasonably be split without obscuring correctness MUST remain atomic, with its
  scope explained in the associated change description. Commit messages MUST be concise and
  state the purpose; commit history MUST make the progression of changes understandable to a
  human. Commit messages MUST NOT include trailers, tags, or boilerplate that add no information
  for a human reader, such as tool-attribution lines (e.g. "Co-Authored-By:" added by an AI
  coding assistant); only include a trailer when it conveys real, reviewer-relevant information
  (e.g. "Fixes #123").
- To work on multiple features in parallel, a contributor MAY create a separate git worktree of
  the `game` repository per feature, named `game_spec_<feature-id>` (e.g.
  `game_spec_009-ci-push-validation`), checked out on its own branch. All quality gates above
  (tests, clippy, fmt, spec traceability) apply to that worktree exactly as they would to `game`
  directly. The branch MUST be merged into `game`'s main branch, with all gates still passing,
  before the feature is considered done; the worktree MAY then be removed. Rationale: isolates
  each feature's working tree and build artifacts so unrelated in-progress specs do not block or
  interfere with one another, without requiring a separate clone per feature.

## Governance

This constitution supersedes other practices. Amendments require a written change to this file
stating rationale, a version bump, and any needed updates to dependent specs, plans, and tasks.
Versioning follows semantic versioning: MAJOR for removed or redefined principles, MINOR for
added principles or materially expanded guidance, PATCH for clarifications. Every plan MUST
include a Constitution Check, and reviews MUST verify compliance; any violation MUST be
justified in the plan's complexity tracking.

**Version**: 1.4.0 | **Ratified**: 2026-10-22 | **Last Amended**: 2026-10-29
