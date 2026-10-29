# Implementation Plan: CI Validation for Pushes and Releases

**Branch**: `009-ci-push-validation` | **Date**: 2026-10-29 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/009-ci-push-validation/spec.md`

## Summary

Add two GitHub Actions workflow files to the game repository (`game/.github/workflows/`):
1. **`ci.yml`** — runs on every push to `main`; builds the Cargo workspace, runs `cargo fmt --check`,
   `cargo clippy --all-targets -- -D warnings` and `cargo test --workspace` (the same gates the
   constitution already requires before merge), and relies on GitHub's built-in failure
   notifications to the pusher — no new external service is introduced.
2. **`release.yml`** — runs only when a version tag (`v*.*.*`) is pushed; rebuilds and re-tests the
   tagged commit, then builds release binaries for the CLI and server crates on Linux, generates a
   changelog with `git-cliff` covering the changes since the previous tag, and publishes a GitHub
   Release (archive + changelog as release notes) while also committing the regenerated
   `CHANGELOG.md` back to `main`. It never runs for ordinary pushes, and it never publishes if the
   build or tests fail.

## Technical Context

**Language/Version**: Rust (stable toolchain, workspace MSRV 1.75 per `game/Cargo.toml`)

**Primary Dependencies**: GitHub Actions (`actions/checkout`, `dtolnay/rust-toolchain`,
`Swatinem/rust-cache`, `softprops/action-gh-release`, `orhun/git-cliff-action`); no new Rust crate
dependencies (git-cliff runs as a standalone CI tool, not a workspace dependency)

**Storage**: N/A

**Testing**: `cargo fmt --check`, `cargo clippy --all-targets -- -D warnings`, `cargo test
--workspace` (the gates the project constitution mandates), plus the project's other existing
test/check suites discovered during implementation — all of which ran in this repository's CI
until it was removed days before this feature, and all of which this feature restores: the
`.githooks/commit-msg` hook's own unit test, the WebAssembly render module build and size guard
(`scripts/build-wasm.sh`), the 175-test web client JS suite (`crates/web/js-tests`, run via `node
--test`), the PTY-based terminal Enter-key integration check (`scripts/pty-enter-check.py`), and
the acceptance-scenario coverage check (`scripts/check-scenarios.sh`, which degrades gracefully to
test-name-only matching when the sibling `specs/` checkout isn't present — see research.md §4)

**Target Platform**: GitHub Actions `ubuntu-latest` runner (project targets Linux first)

**Project Type**: CI/CD automation for an existing Rust Cargo workspace (no application code
changes)

**Performance Goals**: Validation feedback well inside the 10-minute notification budget from
spec SC-002; dependency caching (`Swatinem/rust-cache`) keeps typical runs in the few-minutes range

**Constraints**: Must reuse the build/test/lint commands already mandated by the constitution;
must not require secrets beyond the default `GITHUB_TOKEN`; must not depend on anything outside
the `game` repository (the sibling `specs/` checkout used by `scripts/check-scenarios.sh` is not
guaranteed to be present when the game repo is checked out standalone for CI, so that script is
out of scope for these workflows); the `release.yml` job needs `contents: write` permission on the
default `GITHUB_TOKEN` to commit `CHANGELOG.md` back to `main`, but per spec Edge Cases that commit
MUST be best-effort and MUST NOT block or invalidate an otherwise-successful release

**Scale/Scope**: Exactly two new files, both under `game/.github/workflows/`; no other files
change

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

- **I. Specification-First**: Satisfied — `specs/009-ci-push-validation/spec.md` exists and this
  plan traces to it.
- **II. Test-First & Full Coverage of Game Logic**: Directly reinforced — this feature automates
  the constitution's own requirement that `cargo fmt --check`, `cargo clippy --all-targets -- -D
  warnings` and `cargo test --workspace` pass before merge, turning a manual expectation into an
  enforced, recorded CI gate. No new game logic is added, so no new unit tests are owed by this
  feature itself; the workflow's own correctness is validated via the quickstart scenarios (Phase
  1) rather than `cargo test`, since it is CI configuration, not library code.
- **III. Core/Frontend Separation**: N/A — no application code is touched.
- **IV. Deterministic Simulation**: N/A — no simulation code is touched.
- **V. Four-Player Multiplayer with Bots**: N/A.
- **VI. Dual Interface Parity**: N/A — no gameplay feature is added; both existing interfaces are
  already built by `cargo build --workspace`, which both new workflows invoke identically.
- **Technology & Platform Constraints**: Satisfied — workflows target Linux (`ubuntu-latest`),
  introduce no `unsafe` code, and add no new crate dependencies.
- **Development Workflow & Quality Gates**: Satisfied — this feature is the automation of the
  existing "before merge" gate list, applied to every push to `main`, plus a separate
  tag-triggered release gate that re-runs the same checks before publishing.

No violations. Complexity Tracking is not needed.

## Project Structure

### Documentation (this feature)

```text
specs/009-ci-push-validation/
├── plan.md              # This file (/speckit-plan command output)
├── research.md          # Phase 0 output (/speckit-plan command)
├── data-model.md        # Phase 1 output (/speckit-plan command)
├── quickstart.md        # Phase 1 output (/speckit-plan command)
├── contracts/           # Phase 1 output (/speckit-plan command)
│   ├── ci-workflow.md
│   └── release-workflow.md
└── tasks.md             # Phase 2 output (/speckit-tasks command - NOT created by /speckit-plan)
```

### Source Code (repository root)

```text
game/
├── .github/
│   └── workflows/
│       ├── ci.yml        # NEW — build + full test suite on every push to main
│       └── release.yml   # NEW — build + test + publish a GitHub Release on version tags
├── Cargo.toml             # unchanged — read by both workflows
├── clippy.toml            # unchanged — read by `cargo clippy`
└── crates/                # unchanged — built and tested by both workflows
    ├── core/
    ├── bots/
    ├── net/
    ├── cli/
    ├── web/
    └── web-render/
```

**Structure Decision**: Single project, infrastructure-only change. The two workflow files live in
the `game` repository (the Cargo workspace that actually builds and tests), not in the
spec-tracking `bomberust` repository that only holds specs and a pointer to `game`. No new source
directories, crates, or application code are introduced.

## Complexity Tracking

*No Constitution Check violations — section not needed.*
