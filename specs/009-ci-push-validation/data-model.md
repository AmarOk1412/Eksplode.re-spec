# Phase 1 Data Model: CI Validation for Pushes and Releases

This feature has no application data model (no database, no persisted domain objects). The two
entities identified in the spec map directly onto GitHub's own workflow-run and release concepts;
no new storage is introduced.

## Validation Run

Represents one execution of `ci.yml` for a specific push to `main`.

| Field | Source / Representation | Notes |
|---|---|---|
| Triggering commit | GitHub Actions `push` event SHA | Identifies exactly which push is being validated (FR-005) |
| Start/end time | GitHub Actions run metadata | Visible in the Actions UI run history |
| Result (pass/fail) | Workflow job conclusion | A run fails if build, fmt check, clippy, or any test fails (FR-002) |
| Failure details | Job step logs + GitHub failure notification | Delivered to the pusher without a custom step (FR-003) |

**Lifecycle**: created when a push lands on `main` → runs `fmt`/`clippy`/`build`/`test` steps in
order → concludes success or failure → GitHub records and (on failure) notifies. No manual
transition; a run is never edited after it concludes, only re-triggered by a new push.

## Release

Represents one published, versioned set of build artifacts.

| Field | Source / Representation | Notes |
|---|---|---|
| Version identifier | Git tag matching `v*.*.*` | The explicit "mark this commit as a version" action (FR-009) |
| Source commit | The commit the tag points to | Ties artifacts back to exact code (FR-007) |
| Build artifacts | Archive attached to the GitHub Release | CLI + server binaries built from that commit (FR-006) |
| Changelog | git-cliff output, published twice | As the GitHub Release notes **and** as a committed `CHANGELOG.md` on `main` (FR-011, SC-007) |
| Publication time | GitHub Release `created_at` | Visible to maintainers/users (SC-006) |

**Lifecycle**: created only when a maintainer pushes a matching tag → `release.yml` rebuilds and
re-runs the full test suite against that exact commit → on success, artifacts are built, a
changelog is generated with git-cliff, and a GitHub Release is published with that changelog as
its notes → the workflow then best-effort commits the regenerated `CHANGELOG.md` to `main` (a
failure here does not retract or invalidate the already-published Release); on build/test failure,
nothing is published and the maintainer is notified (FR-008). A release is never produced by an
ordinary push to `main` (FR-009).

## Relationships

A Release is only ever built from a commit that a Validation Run could also validate (same build
and test commands), but the two are independent processes: a Release does not read or depend on a
prior Validation Run's recorded result — it re-runs the checks itself against the tagged commit,
so tagging an already-validated commit is sufficient but the release workflow does not skip its
own checks.
