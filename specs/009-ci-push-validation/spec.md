# Feature Specification: CI Validation for Pushes and Releases

**Feature Branch**: `main` (no feature branch requested)

**Created**: 2026-10-29

**Status**: Draft

**Input**: User description: "Every push must be validated by a CI" — refined with: "Add 2 GitHub CI files for this project, where the first one is done after all push to main where it will build and run every test and send a notification if not working. Second one will produce a release."

## Clarifications

### Session 2026-10-29

- Q: Must the push-validation build always run the entire automated test suite, with no subset, sampling, or fast-path skipping allowed regardless of run time? → A: Yes — all tests must be executed, every time.
- Q: Should producing a release also generate a changelog of the changes included in that version? → A: Yes; generate it with git-cliff.
- Q: Where should the generated changelog be published for each release? → A: Both — the GitHub Release notes/description and a committed `CHANGELOG.md` file in the repository.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Automatic validation of every push to main (Priority: P1)

A contributor pushes commits to the main branch. Without any manual action, the project is built and its full automated test suite is run against that exact commit. If the build or any test fails, the contributor (and the team) is notified so the problem can be fixed quickly.

**Why this priority**: This is the core ask — catching broken builds and failing tests as soon as they land on main, instead of discovering them later or relying on contributors to remember to run checks locally.

**Independent Test**: Push a commit that breaks a test (or the build) to main and confirm a validation run starts automatically, fails, and produces a visible failure notification. Push a commit that is healthy and confirm the run succeeds with no failure notification.

**Acceptance Scenarios**:

1. **Given** a commit is pushed to main, **When** the push is received, **Then** an automated process starts that builds the project and runs its complete automated test suite against that commit.
2. **Given** a validation run in progress, **When** the build or any test fails, **Then** the responsible contributors are notified of the failure, including enough information to identify the failing commit and the failure reason.
3. **Given** a validation run in progress, **When** the build succeeds and all tests pass, **Then** no failure notification is sent and the successful result is recorded and visible to the team.
4. **Given** a past push to main, **When** anyone looks up its validation result, **Then** they can see whether it passed or failed and when it ran.

---

### User Story 2 - Produce a release (Priority: P2)

A maintainer wants to publish a new version of the project. Triggering a release kicks off an automated process that builds the project and produces the artifacts needed to distribute that version, without the maintainer having to perform manual build steps.

**Why this priority**: Releases matter less often than everyday push validation, but once a version is ready, producing it reliably and repeatably (rather than by hand) avoids mistakes and saves maintainer time. It depends on User Story 1 only in spirit (a release should only be built from code that has already passed validation), not technically.

**Independent Test**: Trigger a release for a specific version and confirm that downloadable release artifacts for that version are produced automatically and associated with that version, without manual build steps.

**Acceptance Scenarios**:

1. **Given** a maintainer marks a specific commit as a new version (e.g., by tagging it), **When** the release process runs, **Then** it builds that commit and produces downloadable release artifacts labeled with that version.
2. **Given** a release is being produced, **When** the underlying build or tests fail, **Then** no release artifacts are published and the maintainer is notified of the failure.
3. **Given** a release has been produced, **When** a maintainer or user looks for it, **Then** they can find the version's artifacts and know which commit they came from.
4. **Given** an ordinary push to main that is not marked as a new version, **When** that push is validated, **Then** the release process does not run for it.

---

### Edge Cases

- What happens when a push to main includes only non-code changes (e.g., documentation)? Validation still runs so the recorded history of main stays consistent.
- What happens when several pushes to main happen in quick succession? Each push gets its own recorded validation result; a later push does not hide the failure of an earlier one.
- What happens when a release is triggered for a commit whose push validation has not run or has failed? The release process MUST NOT publish artifacts for that commit; the maintainer is notified instead.
- What happens when the notification channel itself is unavailable? The validation result MUST still be recorded and visible even if the notification cannot be delivered.
- How does the system handle a validation run that never finishes (e.g., a hang)? It MUST be treated as a failure after a bounded maximum run time rather than blocking future pushes indefinitely.
- What happens when the release process needs to commit the updated `CHANGELOG.md` back to the repository? It MUST commit that update to the main branch without altering or moving the version tag itself, and that commit MUST NOT be required for the release's published artifacts or notes to be considered complete (so a failure to commit the file back does not block or invalidate the release).

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: System MUST automatically build the project and run the entire automated test suite — every test, with no subset, sampling, or fast-path skipping — whenever new commits are pushed to the main branch, without requiring a person to start the run manually.
- **FR-002**: System MUST treat a push to main as failed validation if the build does not succeed or if any test in the automated test suite fails.
- **FR-003**: System MUST notify the relevant contributors when a push to main fails validation, with enough detail to identify the failing commit and the cause of failure.
- **FR-004**: System MUST NOT send a failure notification for a push to main that builds successfully and passes all tests.
- **FR-005**: System MUST retain, for every push to main, a record of whether validation passed or failed and when it ran, visible to the team after the fact.
- **FR-006**: System MUST provide a separate release process, triggered independently from ordinary pushes, that builds the project and produces distributable release artifacts for a specific version.
- **FR-007**: System MUST associate each set of release artifacts with the version and commit it was built from.
- **FR-008**: System MUST NOT publish release artifacts if the build or test suite fails during the release process; it MUST instead notify the maintainer of the failure.
- **FR-009**: System MUST NOT run the release process for ordinary pushes to main that are not explicitly marked as a new version.
- **FR-010**: System MUST bound how long a single validation or release run is allowed to take, treating a run that exceeds that bound as a failure.
- **FR-011**: System MUST generate a changelog summarizing the changes included in each release, using git-cliff, and publish it in two places: as the published release's notes/description, and as an updated `CHANGELOG.md` file committed to the repository.

### Key Entities

- **Validation Run**: One execution of the build-and-test process for a specific pushed commit on main. Key attributes: triggering commit, start/end time, pass/fail result, failure details (if any).
- **Release**: A published, versioned set of distributable artifacts built from a specific commit. Key attributes: version identifier, source commit, build artifacts, changelog of included changes, publication time.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: 100% of pushes to the main branch automatically start a validation run with no manual action required.
- **SC-002**: Contributors are notified of a failing push to main within 10 minutes of the push, with no manual steps needed to trigger the notification.
- **SC-003**: Zero failure notifications are sent for pushes to main that actually build successfully and pass all tests (no false-positive noise).
- **SC-004**: 100% of past pushes to main have a retrievable pass/fail record available to the team.
- **SC-005**: Producing a release requires a single triggering action from a maintainer (e.g., tagging a version) and completes without any manual build steps.
- **SC-006**: 100% of published releases are traceable to the exact commit and version they were built from.
- **SC-007**: 100% of published releases include a changelog listing the changes introduced since the previous release.

## Assumptions

- "Every push" refers to pushes that update the main branch, since that is the integration point where breakage affects every contributor; validation of other branches or in-flight pull requests is not required by this feature.
- Notification means reaching the people responsible for the breakage (e.g., the author of the failing commit and/or the team) through whatever standard mechanism the hosting platform already offers; no specific channel (email, chat, etc.) is mandated.
- A release is triggered by explicitly marking a commit as a new version (e.g., tagging it), not by every push to main.
- "Produces a release" means building distributable artifacts for the project's existing interfaces and publishing them in a place where they are associated with their version; it does not imply any new distribution channel beyond what the project already uses for distributing builds.
- The automated test suite and build referenced here are the same ones contributors are already expected to run before merging (full test suite, lint/format checks), per the project's existing quality bar.
- The changelog's scope is the changes since the previous release (git-cliff's standard behavior), not the entire project history.
- The `CHANGELOG.md` commit is written back to the main branch using the release automation's own identity; it is a convenience record of the release and is not itself validated as a new push requiring a fresh release.
