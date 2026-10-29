# Contract: `ci.yml` (push validation workflow)

This is the external interface contributors and maintainers observe: a GitHub Actions workflow,
not an HTTP API. The "contract" is its trigger, guaranteed steps, and observable outcomes.

## Trigger

- **Event**: `push`
- **Branches**: `main` only
- Does **not** trigger on pushes to any other branch, and does **not** trigger on pull requests
  (see research.md §2).

## Guaranteed steps (in order)

1. Check out the pushed commit.
2. Install the stable Rust toolchain with the `rustfmt` and `clippy` components, plus the
   `wasm32-unknown-unknown` target.
3. Install Node.js (for the web client's JS test suite).
4. Restore/populate the dependency cache.
5. Run `cargo fmt --all -- --check`.
6. Run `cargo clippy --workspace --all-targets -- -D warnings`.
7. Run `sh .githooks/test-commit-msg.sh` (the commit-msg hook's own unit test).
8. Run `cargo build --workspace`.
9. Run `cargo test --workspace`.
10. Run `sh scripts/build-wasm.sh` (builds and size-checks the WebAssembly render module).
11. Run `node --test` in `crates/web/js-tests` (the web client's JS test suite).
12. Run `python3 scripts/pty-enter-check.py` (terminal Enter-key integration check).
13. Run `sh scripts/check-scenarios.sh` (acceptance-scenario coverage check).

Each step runs only if the previous step succeeded (default GitHub Actions job behavior: a failing
step stops the job). Any failing step marks the whole run as failed. This is the project's full
original test suite (restored after being removed along with unrelated legacy files two days
before this feature was added — see research.md §4), not just `cargo test`.

## Observable outcomes

| Outcome | Guarantee |
|---|---|
| All steps succeed | Run shows green in the Actions UI; commit shows a success status; **no** failure notification is sent (FR-004) |
| Any step fails | Run shows red in the Actions UI; commit shows a failure status; GitHub's built-in notification reaches the pusher (FR-003) |
| Run exceeds the configured job timeout | Treated as a failed run (FR-010) |

## Non-goals

- Does not publish any artifact.
- Does not run for non-`main` branches or pull requests.
