# Quickstart: Validating CI Push Validation and Release Workflows

## Prerequisites

- Push access to `AmarOk1412/Eksplode.re` (the `game` repository), or a fork for a dry run.
- Rust stable toolchain locally (optional, to reproduce the same checks before pushing):
  `rustup component add rustfmt clippy`.

## Validating `ci.yml` (push to main)

1. Locally reproduce the exact gate the workflow runs, from the `game/` directory:
   ```sh
   cargo fmt --all -- --check
   cargo clippy --workspace --all-targets -- -D warnings
   cargo build --workspace
   cargo test --workspace
   ```
   All four MUST succeed locally before relying on CI as the first check.
2. Push a commit to `main` (or, for a dry run, to a fork's `main`).
3. Open the repository's **Actions** tab and confirm a `ci.yml` run started automatically for that
   commit (see contracts/ci-workflow.md).
4. **Success case**: confirm the run finishes green and no failure notification is received.
5. **Failure case**: push a commit that deliberately breaks a test (e.g., temporarily flip an
   assertion), push it to a fork's `main`, confirm the run finishes red, and confirm a failure
   notification is received by the pusher. Revert the deliberate breakage afterward.
6. Confirm the run's pass/fail result and timestamp remain visible in the Actions run history
   after the fact (FR-005).

## Validating `release.yml` (tag push)

1. On a commit that already passes `ci.yml`, create and push a test tag:
   ```sh
   git tag v0.0.0-test
   git push origin v0.0.0-test
   ```
   (Use a fork or a clearly-marked test tag so it does not collide with a real release.)
2. Confirm `release.yml` starts automatically (and `ci.yml` does **not** start again for the tag
   push itself, since it only triggers on branch pushes to `main`).
3. Confirm the workflow re-runs `fmt`/`clippy`/`build`/`test` against the tagged commit.
4. **Success case**: confirm a GitHub Release named `v0.0.0-test` appears with the packaged
   binaries attached, and that it links back to the source commit.
5. Confirm the Release notes contain a git-cliff-generated changelog of commits since the previous
   tag, and confirm `CHANGELOG.md` on `main` was updated with the same content in a follow-up
   commit (FR-011, SC-007).
6. **Failure case**: tag a commit that fails a test, confirm no Release is created and the
   maintainer is notified.
7. Delete the test tag, any test Release, and revert the test `CHANGELOG.md` commit afterward:
   ```sh
   git push --delete origin v0.0.0-test
   git tag -d v0.0.0-test
   ```

## Expected outcome

Both workflows behave exactly as described in `contracts/ci-workflow.md` and
`contracts/release-workflow.md`, satisfying spec.md's acceptance scenarios and success criteria
(SC-001 through SC-006) without any manual build or notification step.
