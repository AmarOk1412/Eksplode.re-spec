# Contract: `release.yml` (release workflow)

## Trigger

- **Event**: `push`
- **Tags**: matching `v*.*.*` (e.g., `v0.2.0`)
- Does **not** trigger on ordinary pushes to `main` or any branch (FR-009).

## Guaranteed steps (in order)

1. Check out the exact commit the tag points to (full history, `fetch-depth: 0`, so git-cliff can
   see all prior tags and commits).
2. Install the stable Rust toolchain with the `rustfmt` and `clippy` components, plus the
   `wasm32-unknown-unknown` target.
3. Install Node.js.
4. Restore/populate the dependency cache.
5. Run the same full gate as `ci.yml` against the tagged commit — `cargo fmt --all -- --check`,
   `cargo clippy --workspace --all-targets -- -D warnings`, `sh .githooks/test-commit-msg.sh`,
   `cargo build --workspace`, `cargo test --workspace`, `sh scripts/build-wasm.sh`, `node --test`
   (in `crates/web/js-tests`), `python3 scripts/pty-enter-check.py`, `sh
   scripts/check-scenarios.sh` (the release never skips these checks, even if the same commit
   already passed `ci.yml` on `main` — see data-model.md "Relationships").
6. Build release binaries: `cargo build --workspace --release`.
7. Package the `eksplode-re-cli` and `eksplode-re-server` binaries (plus the committed WebAssembly
   render module and web assets) into a single archive named with the tag.
8. Generate release notes with `git-cliff --latest` (the commits belonging to this tag specifically
   — not `--unreleased`, which is always empty here since the tag already exists on `HEAD` by the
   time this tag-triggered workflow runs; see research.md §8) and a full `CHANGELOG.md` with
   `git-cliff` (default mode, covering every tagged release to date).
9. Publish a GitHub Release for the tag with the archive attached and the `--latest` changelog as
   the release notes/description.
10. Best-effort commit the regenerated `CHANGELOG.md` to `main` using the workflow's own identity,
    staging the file before diffing so a first-ever `CHANGELOG.md` is correctly detected as a
    change (a plain `git diff` never flags an untracked file — see research.md §8).

If any of steps 5–8 fails, steps 9 and 10 do not run. If step 10 fails after step 9 succeeded, the
Release published in step 9 remains valid and complete — the `CHANGELOG.md` commit is not a
precondition for a successful release (per spec Edge Cases).

## Observable outcomes

| Outcome | Guarantee |
|---|---|
| All steps succeed | A GitHub Release exists, named after the tag, with the archive and changelog attached, traceable to the source commit, and `CHANGELOG.md` on `main` reflects the same changelog (FR-006, FR-007, FR-011, SC-006, SC-007) |
| Any check, build, or changelog-generation step fails | No Release is created/updated; the maintainer who pushed the tag is notified via GitHub's built-in failure notification (FR-008) |
| Release notes published but the `CHANGELOG.md` push-back fails (e.g., protected branch) | The Release itself remains published and valid; only the repository-committed copy of the changelog is missing until retried |
| Ordinary push to `main` (no tag) | Workflow does not run at all (FR-009) |

## Non-goals

- Does not build for platforms other than Linux (`ubuntu-latest`) in this iteration.
- Does not publish to any channel other than GitHub Releases (no Docker registry, no package
  registry).
- Does not infer a version automatically — the tag itself is the version marker.
- Does not retroactively regenerate `CHANGELOG.md` for past releases — only applies going forward
  from this feature's first tagged release.
