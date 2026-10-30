# Quickstart: Bot Behaviours and Difficulty Levels

Validation guide. Entities and rules: [data-model.md](data-model.md); wire/flag contract:
[contracts/room-settings.md](contracts/room-settings.md); API/UI contract:
[contracts/bots-and-ui.md](contracts/bots-and-ui.md).

## Prerequisites

Run everything from `game/`. Rust stable, Node (for JS tests), and the WebAssembly build from
`scripts/build-wasm.sh` for the browser check.

## Automated checks

```sh
cargo fmt --check
cargo clippy --all-targets -- -D warnings
cargo test --workspace
cargo test -p eksplode-re-bots --test soak -- --nocapture   # prints SC-001..SC-004 statistics
(cd crates/web/js-tests && node --test)
```

Baseline (2026-10-30, before this feature): fmt, clippy, `cargo test --workspace` and the 199 JS
tests all passed; no pre-existing failures.

Expected: all pass; soak output shows each behaviour picked 25–42% (SC-001), Hard beats Easy ≥ 65%,
Medium beats Easy ≥ 55%, Hard beats Medium ≥ 55% (SC-002), Easy self-eliminations ≥ 2× Hard (SC-003),
Aggressive bombs-on-opponent ≥ 1.3× Protective and Protective survives longer (SC-004).

## Manual checks

1. **Terminal host**: `cargo run -p eksplode-re-cli -- host --bots 3 --bot-difficulty easy` → bots
   are visibly slower and sometimes take wrong turns; repeat with `hard` → bots react at once.
   `local --bot-difficulty easy` is refused (usage error): only the host chooses.
2. **Terminal host setup**: `cargo run -p eksplode-re-cli -- host --bots 3` → setup screen shows
   "Bot difficulty < Medium >"; change it, create the room; the lobby line ends with `- Hard bots`.
   In the lobby, the owner presses `d` to cycle; a joined non-owner pressing `d` gets the owner notice.
3. **Web**: `cargo run -p eksplode-re-web -- --bind 127.0.0.1` and open the page; create a room → the
   form shows "Bot difficulty" preselected Medium; in the lobby the owner can change it in the
   configuration panel; a second browser (non-owner) sees the value but cannot change it; during a
   match the setting cannot be changed.
4. **Compatibility**: send a `CreateRoom` whose settings omit `bot_difficulty` (e.g. with an existing
   `net` test helper) → the room reports `"bot_difficulty":"medium"`.
5. **Developer mode**: `cargo run -p eksplode-re-web -- --bind 127.0.0.1 --dev`; as owner open the
   developer panel, set "Bot behaviour" to Aggressive and start → every bot charges opponents; a
   non-owner sees the value but cannot change it; without `--dev` the panel and command are absent.

## Done criteria

- Every acceptance scenario of [spec.md](spec.md) maps to a named test (`f12_usN_asM_*`).
- README, `docs/GAMEPLAY.md`, `docs/PROTOCOL.md` and `docs/DEVELOPMENT.md` document the setting,
  the host flag, the `d` key, the three behaviours and the developer behaviour override.
