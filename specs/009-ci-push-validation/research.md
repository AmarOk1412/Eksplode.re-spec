# Phase 0 Research: CI Validation for Pushes and Releases

No `NEEDS CLARIFICATION` markers remain in the Technical Context, but the following decisions
needed an explicit choice among reasonable alternatives. Each is recorded here per spec Assumptions.

## 1. Where the workflows live

**Decision**: Add `ci.yml` and `release.yml` under `game/.github/workflows/`, in the
`Eksplode.re` (game) repository.

**Rationale**: GitHub Actions only runs workflows that live in the repository being pushed to.
The buildable Cargo workspace (crates, `Cargo.toml`, tests) lives in `game`, which is checked out
into this spec-tracking workspace as a submodule-style gitlink pointing at the `Eksplode.re`
repository (`git@github.com:AmarOk1412/Eksplode.re.git`). The outer `bomberust`
(`Eksplode.re-spec`) repository contains no buildable code, so a workflow placed there would have
nothing to build or test.

**Alternatives considered**: Placing workflows in the outer spec repository and having them `cd`
into the submodule — rejected because pushes that trigger validation happen against the game
repository's own `main` branch, not the spec repository's.

## 2. Push-validation trigger scope

**Decision**: `ci.yml` triggers on `push` to `main` only.

**Rationale**: Matches the explicit instruction ("done after all push to main") and spec FR-001 /
Assumptions, which scope "every push" to the main branch specifically.

**Alternatives considered**: Also triggering on `pull_request` — a common and valuable addition,
but explicitly out of scope per the spec's Assumptions section ("validation of other branches or
in-flight pull requests is not required by this feature"); left for a future feature if needed.

## 3. Failure notification mechanism

**Decision**: Rely on GitHub's built-in workflow-failure notifications (GitHub notifies the user
whose push triggered a failing run, via their existing notification settings/email) rather than
adding a custom notification step.

**Rationale**: Satisfies FR-003/SC-002 (contributors notified of failure, no manual steps) and
FR-004 (no notification on success, since GitHub does not notify on success by default) with zero
additional secrets, services, or workflow steps. Matches spec Assumptions: "no specific channel
... is mandated."

**Alternatives considered**: A Slack/Discord webhook step — rejected because it would require
provisioning and storing a new secret and an external channel that is not part of the project
today, which the spec's Assumptions explicitly avoid ("does not imply any new distribution
channel"). An email step via a dedicated action — rejected as redundant with GitHub's native
notification and adds unnecessary secret handling.

## 4. Build/test/lint commands used by `ci.yml` (revised after discovering the project's prior CI)

**Decision**: `cargo fmt --all -- --check`, `cargo clippy --workspace --all-targets -- -D
warnings`, `sh .githooks/test-commit-msg.sh`, `cargo build --workspace`, `cargo test --workspace`,
`sh scripts/build-wasm.sh`, `node --test` (in `crates/web/js-tests`), `python3
scripts/pty-enter-check.py`, and `sh scripts/check-scenarios.sh`, in that order.

**Rationale**: While implementing and live-validating this feature (pushing to the real
`AmarOk1412/Eksplode.re` repository), `git log -- .github/workflows` showed this project already
had a CI workflow until it was deleted two days earlier in an unrelated cleanup commit (`core:
remove old files`, 2026-10-27). That workflow ran all of the commands above, not just
`fmt`/`clippy`/`cargo test`. Given FR-001's explicit requirement that "all tests must be executed,"
restoring the full original set is correct, not optional scope creep. All of them were verified to
still pass locally (`cargo test --workspace`: hundreds of tests; `node --test`:
175 tests; `pty-enter-check.py` and `build-wasm.sh`: both pass) before being added back.

**Alternatives considered (superseded)**: An earlier version of this research document excluded
`scripts/check-scenarios.sh`, reasoning that it reads manual checklists from the sibling `specs/`
repository via relative paths (`../specs/...`) not present in a standalone `game` checkout. That
reasoning was incorrect: the script's `in_manual` helper explicitly returns false when the
checklist file doesn't exist (`[ -f "$2" ] || return 1`), so it degrades gracefully to test-name-only
matching — confirmed by running it with the manual-checklist env vars pointed at nonexistent paths,
which still printed "all scenarios covered". It is included in both workflows.

## 5. Release trigger and versioning

**Decision**: `release.yml` triggers on `push` of tags matching `v*.*.*` (semantic version tags).

**Rationale**: Tag-triggered releases are the standard, low-friction way to mark "this commit is
a version" without adding a manual-dispatch UI step or a new file-based version marker. Matches
spec FR-009 (release process must not run for ordinary pushes) and Assumptions ("triggered by
explicitly marking a commit as a new version, e.g., tagging it").

**Alternatives considered**: `workflow_dispatch` (fully manual trigger) — viable but adds an extra
manual action beyond "push a tag"; `workflow_dispatch` can still be added later without changing
this decision. A version file bump detected on `main` — rejected as more complex and harder to
make idempotent (re-running on the same version).

## 6. Release artifacts

**Decision**: Build release binaries for the `eksplode-re-cli` and `eksplode-re-web` crates on
`ubuntu-latest` (`cargo build --workspace --release`), package them into a single archive, and
publish them as assets on a GitHub Release named after the tag, using the already-committed
WebAssembly render module (per README, it is checked into the repo and does not need rebuilding
for a release unless `scripts/build-wasm.sh` output changed — rebuilding it is a reasonable
belt-and-suspenders step the workflow performs via the existing script).

**Rationale**: Matches spec FR-006/FR-007 (distributable artifacts tied to a version/commit) and
Assumptions ("no new distribution channel beyond what the project already uses"). Linux-only
matches the constitution's "Target platform: Linux first."

**Alternatives considered**: Multi-platform (Windows/macOS) binary matrix — reasonable future
enhancement, not required by the spec and adds build-matrix complexity beyond what was asked;
Docker image publishing — no existing Dockerfile or registry convention in the project, so it
would introduce a new distribution channel the Assumptions section says to avoid.

## 7. Changelog generation and destination

**Decision**: Generate the changelog with `git-cliff` (via `orhun/git-cliff-action`), scoped to
commits since the previous tag (git-cliff's default range behavior), and publish it in both
places confirmed during `/speckit-clarify`: as the GitHub Release's notes/description, and as a
regenerated `CHANGELOG.md` committed back to `main` using the workflow's own identity (e.g.
`github-actions[bot]`) with `contents: write` permission on the default `GITHUB_TOKEN`.

**Rationale**: Directly implements FR-011/SC-007 and the explicit clarification answer "C — both".
Using the project's existing commit convention (`.githooks/commit-msg`'s `part: short desc`
format, enforced since feature 001) as git-cliff's commit-parsing input needs no new commit
convention — git-cliff can group entries by the existing `part:` prefixes. The `CHANGELOG.md`
commit is explicitly non-blocking per spec Edge Cases: if the push-back fails (e.g., a protected
branch rule), the GitHub Release itself — artifacts and notes — must still be considered
complete and published.

**Alternatives considered**: Changelog in release notes only (original Option A) — rejected once
the user picked "both" during clarification; a separate changelog-hosting service or wiki page —
rejected as a new distribution channel the spec's Assumptions explicitly avoid; scoping the
changelog to full project history on every release — rejected as noisy and not git-cliff's
standard behavior, and not what "since the previous release" in SC-007 implies.

## 8. Two bugs found during live validation, and why the fixes are correct

Live-testing `release.yml` against a real `v0.0.0-test` tag on `AmarOk1412/Eksplode.re` surfaced
two defects invisible in local dry runs (which never had a real pre-existing tag at `HEAD`):

- **Empty release notes**: the release-notes step used `git-cliff --unreleased`, i.e. "commits not
  yet covered by any tag." Because this workflow triggers on a tag *push* — the tag already exists
  on `HEAD` by the time the job runs — there are never any such commits, so the body was always
  empty. `git-cliff --current` fails outright ("No tag exists for the current commit") in this same
  already-tagged scenario; `git-cliff --latest` ("commits belonging to the most recent tag") is the
  flag that actually works when the trigger model is "the tag already exists," confirmed by
  reproducing the real repository's tag/history locally. This is now what `release.yml` uses.
- **`CHANGELOG.md` was never committed**: the push-back step guarded on `git diff --quiet --
  CHANGELOG.md`, but `git diff` only compares already-tracked content; for a file that has never
  existed in the repository before (this was the project's first-ever `CHANGELOG.md`), that check
  always reports "no difference" even though the file is new and unstaged. Staging the file first
  (`git add`) and checking `git diff --cached --quiet` instead correctly detects brand-new files as
  well as modified ones.

Both fixes were verified against the real repository with a second disposable tag
(`v0.0.1-test`): the Release body contained the real per-release changelog, and `CHANGELOG.md`
landed on `main` in a `docs: update CHANGELOG.md for v0.0.1-test` commit. The test tags, their
GitHub Releases (which GitHub removes automatically once the backing tag is deleted), and the test
`CHANGELOG.md` commit (reverted) were all cleaned up afterward.
