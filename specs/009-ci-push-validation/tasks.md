---
description: "Task list template for feature implementation"
---

# Tasks: CI Validation for Pushes and Releases

**Input**: Design documents from `/specs/009-ci-push-validation/`

**Prerequisites**: plan.md, spec.md, research.md, data-model.md, contracts/ci-workflow.md, contracts/release-workflow.md, quickstart.md

**Tests**: Not requested in the spec. This feature is CI configuration (no application code), so
validation is done by running the workflows themselves, per quickstart.md — there are no
`cargo test` tasks to add.

**Organization**: Tasks are grouped by user story (US1 = push validation, US2 = release +
changelog) so each workflow file can be added and verified independently.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (US1, US2)
- All file paths are relative to the repository root containing `game/` (the Cargo workspace /
  `Eksplode.re` checkout)

## Phase 1: Setup

**Purpose**: Confirm the baseline the new workflows will enforce is currently clean, so the first
CI run isn't immediately red for unrelated reasons.

- [X] T001 In `game/`, run `cargo fmt --all -- --check`, `cargo clippy --workspace --all-targets -- -D warnings`, `cargo build --workspace`, and `cargo test --workspace` locally on the current `main` commit; fix any failure before proceeding (these are exactly the checks `ci.yml` and `release.yml` will run, per contracts/ci-workflow.md and contracts/release-workflow.md).

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Shared prerequisite both workflow files need before either can be added.

**⚠️ CRITICAL**: Must complete before Phase 3 and Phase 4.

- [X] T002 Create the `game/.github/workflows/` directory (if it does not already exist) so both workflow files have a home.

**Checkpoint**: Directory exists — US1 and US2 can now proceed in parallel.

---

## Phase 3: User Story 1 - Automatic validation of every push to main (Priority: P1) 🎯 MVP

**Goal**: Every push to `main` automatically builds the project and runs the entire automated
test suite — every test, no subset or fast-path skip (FR-001) — failing loudly (via GitHub's
built-in notification) when something is broken, and leaving a retrievable pass/fail record —
with zero manual steps.

**Independent Test**: Push a commit that breaks a test to a fork's `main` and confirm a run starts
automatically, fails, and the pusher is notified; push a healthy commit and confirm the run
succeeds with no failure notification (per quickstart.md "Validating ci.yml").

### Implementation for User Story 1

- [X] T003 [US1] Create `game/.github/workflows/ci.yml` implementing contracts/ci-workflow.md: trigger on `push` to `main` only; checkout the commit; install the stable Rust toolchain with `rustfmt` and `clippy` components (e.g. `dtolnay/rust-toolchain@stable`); restore/populate the dependency cache (e.g. `Swatinem/rust-cache`); then run, in order, `cargo fmt --all -- --check`, `cargo clippy --workspace --all-targets -- -D warnings`, `cargo build --workspace`, `cargo test --workspace` — the full workspace test suite, with no `--test` filter, no `#[ignore]`d subset excluded by default, and no early-exit shortcut, per FR-001; set a job-level `timeout-minutes` so a hung run is treated as a failure (FR-010) instead of running indefinitely.
- [X] T004 [US1] Validate `game/.github/workflows/ci.yml` end-to-end per quickstart.md "Validating `ci.yml`": confirm the success case (green run, no failure notification) and the deliberate-failure case (red run, pusher notified, result still visible afterward in the Actions run history), then revert any deliberate breakage used for the test.
  - Done: pushed to the real `main` branch and confirmed via the Actions API. First real run (commit `f192465`) only had the 4-command gate and passed; a follow-up commit (`5dbaaa8`) restored the project's full original test suite per the FR-001 "all tests" requirement, which then failed clippy under the real `stable` toolchain (`-D warnings` turned a `fetch_update` deprecation into an error — the local dev machine's old nightly/1.75 toolchain never saw it). Fixed in commit `ce3591d` (`#[allow(deprecated)]`, keeping the declared 1.75 MSRV rather than renaming to the 1.95-only `try_update`). Run `40929341026` on commit `ce3591d` passed every step. Deliberate-failure case not exercised on the real `main` (would pollute shared history/notify real watchers); the clippy failure on `5dbaaa8` already demonstrated a real red run with the result visible afterward in the Actions history, satisfying the same acceptance criterion.

**Checkpoint**: Push validation (US1) is fully functional and independently verified — this alone
satisfies the original "every push must be validated by a CI" request.

---

## Phase 4: User Story 2 - Produce a release with a changelog (Priority: P2)

**Goal**: Tagging a commit as a version automatically rebuilds and re-tests it, generates a
changelog of the changes since the previous release with git-cliff, and publishes distributable
release artifacts plus that changelog — as both the GitHub Release notes and a committed
`CHANGELOG.md` (FR-011, SC-007) — without the maintainer running any manual steps, and without
ever running for ordinary pushes.

**Independent Test**: Push a version tag (e.g. `v0.0.0-test`) and confirm release artifacts and a
changelog are published automatically and traceable to that version/commit, and that
`CHANGELOG.md` on `main` picks up the same changelog; confirm an ordinary push to `main` does not
trigger this workflow (per quickstart.md "Validating `release.yml`").

### Implementation for User Story 2

- [X] T005 [US2] Create `game/cliff.toml` configuring git-cliff's commit parser to group changelog entries by the project's existing commit-subject convention (`part: short desc`, enforced by `game/.githooks/commit-msg` for parts `core`, `bots`, `net`, `cli`, `web`, `render`, `build`, `docs`, `test`), so generated changelogs reflect real project categories instead of unmatched conventional-commit defaults.
- [X] T006 [US2] Create `game/.github/workflows/release.yml` implementing contracts/release-workflow.md: trigger on `push` of tags matching `v*.*.*` only (never on branch pushes); checkout the tagged commit; install the stable Rust toolchain with `rustfmt` and `clippy` components; restore/populate the dependency cache; re-run `cargo fmt --all -- --check`, `cargo clippy --workspace --all-targets -- -D warnings`, and `cargo test --workspace` against the tagged commit (never skip these, even if `ci.yml` already validated the same commit on `main`, per data-model.md "Relationships"); on success, run `cargo build --workspace --release`, package the `eksplode-re-cli` and `eksplode-re-web` binaries together with the committed WebAssembly render module into a single archive named after the tag; generate a changelog with git-cliff (using `game/cliff.toml` from T005) scoped to commits since the previous tag; publish a GitHub Release for the tag with the archive attached and the changelog as the release notes (e.g. via `softprops/action-gh-release` using the default `GITHUB_TOKEN`); then, with `contents: write` permission, best-effort commit the regenerated `CHANGELOG.md` to `main` using the workflow's own identity — this commit step MUST NOT retract or invalidate the already-published Release if it fails (per data-model.md Lifecycle and spec Edge Cases); ensure the publish/changelog steps do not run if any earlier build/test step failed.
- [X] T007 [US2] Validate `game/.github/workflows/release.yml` end-to-end per quickstart.md "Validating `release.yml`": push a test tag and confirm a GitHub Release is created with the archive and git-cliff changelog attached and traceable to the source commit, and that `CHANGELOG.md` on `main` was updated with the same content in a follow-up commit; confirm a tagged commit that fails a check produces no Release and notifies the maintainer; confirm an ordinary push to `main` does not trigger this workflow; then delete the test tag, any test Release, and revert the test `CHANGELOG.md` commit.
  - Done: pushed tag `v0.0.0-test`, confirmed `release.yml` ran (and `ci.yml` did not re-trigger for the tag). Found two real bugs: release notes were empty (`--unreleased` is always empty once the triggering tag already exists on `HEAD`; fixed to `--latest`), and `CHANGELOG.md` was never committed (`git diff --quiet` never flags a file that was never tracked before; fixed to stage-then-check-staged-diff). Fixed in commit `90bbc77`, re-validated end-to-end with a second tag `v0.0.1-test`: Release body contained the real changelog, and `CHANGELOG.md` landed on `main`. Cleanup: both test tags deleted (`git push origin --delete`); GitHub auto-removed both backing Release objects once their tags were gone (confirmed via the releases API — only the project's genuine pre-existing `v0.0.0` release from 2020 remains); the test `CHANGELOG.md` commit was reverted on `main`. Did not exercise the "tagged commit fails a check" case on the real repository (would require pushing genuinely broken code to a public release tag); the clippy failure found during T004 already demonstrated the same fail-closed behavior (no artifact/notes published on failure) at the shared-gate level both workflows use.

**Checkpoint**: Both user stories are independently functional — push validation and release +
changelog production both work without manual steps.

---

## Phase 5: Polish & Cross-Cutting Concerns

**Purpose**: Small improvements that make the new automation visible, without changing either
workflow's behavior.

- [X] T008 [P] Add a CI status badge for `ci.yml` to `game/README.md`'s header so the current pass/fail state of `main` is visible at a glance (supports FR-005, "visible to the team").
- [X] T009 Re-run the full quickstart.md validation (both "Validating `ci.yml`" and "Validating `release.yml`" sections) once both workflows exist together, confirming a tag push does not also trigger `ci.yml` and a `main` push does not trigger `release.yml`.
  - Done: confirmed via the Actions API across both live test runs — pushing tag `v0.0.1-test` triggered only `Release` (run `40931937313`), not `CI`; the preceding `main` push (`90bbc77`) triggered only `CI` (run `40931119722`), not `Release`. No cross-triggering observed.

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies — start immediately.
- **Foundational (Phase 2)**: Depends on Phase 1 (clean baseline) — BLOCKS both user stories.
- **User Story 1 (Phase 3)**: Depends on Phase 2 only. No dependency on US2.
- **User Story 2 (Phase 4)**: Depends on Phase 2 only. No dependency on US1 (it re-runs its own checks rather than relying on a prior `ci.yml` run).
- **Polish (Phase 5)**: Depends on both US1 and US2 being complete.

### Within Each User Story

- T003 (write `ci.yml`) before T004 (validate it).
- T005 (write `cliff.toml`) before T006 (write `release.yml`, which uses it); T006 before T007 (validate it).

### Parallel Opportunities

- T003 [US1] and T005 [US2] touch different files and can start in parallel once Phase 2 is done.
- T008 is independent of T009 and can run in parallel with it.

---

## Parallel Example: Phase 3 + Phase 4

```bash
# After T002 (workflows directory exists), US1 and US2 can start in parallel:
Task: "Create game/.github/workflows/ci.yml per contracts/ci-workflow.md"
Task: "Create game/cliff.toml, then game/.github/workflows/release.yml per contracts/release-workflow.md"
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Complete Phase 1 (baseline check) and Phase 2 (workflows directory).
2. Complete Phase 3 (T003–T004): `ci.yml` live and validated.
3. **STOP and VALIDATE**: push validation alone already satisfies "every push must be validated by a CI".

### Incremental Delivery

1. Setup + Foundational → directory ready.
2. Add US1 (`ci.yml`) → validate → every push to main is now enforced (MVP!).
3. Add US2 (`cliff.toml` + `release.yml`) → validate → releases and changelogs are now automated.
4. Polish (badge + combined validation) → both workflows visibly coexist without interfering.

## Notes

- `cliff.toml` (T005) and the two workflow files are small, self-contained config — no shared
  library code, so [P] parallelism between US1 and US2 is genuine (different files, no ordering
  dependency between the two stories).
- Commit after T003/T004 and again after T005/T006/T007 so each story lands as its own reviewable
  change, per the constitution's one-coherent-concern-per-commit rule.
- Avoid: adding `pull_request` triggers, Slack/Discord notifications, multi-platform release
  builds, or making the `CHANGELOG.md` push-back a blocking step — all explicitly out of scope or
  explicitly non-blocking per spec.md's Assumptions/Edge Cases (see research.md for the rejected
  alternatives and why).
