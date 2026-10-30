# Quickstart: Validating the Menu/Lobby/Customize/Scoreboard Reorganization

## Prerequisites

- `cargo build --workspace` succeeds.
- WASM renderer rebuilt via `scripts/build-wasm.sh` (picks up `web-render` Look/Costume changes).
- Local web server running (`cargo run -p eksplode-re-web` or the project's existing dev-run script).

## Core/net unit tests (write failing first, per Principle II)

1. `crates/core`: `Lobby::remove_bot_at` removes the targeted bot and errors on a non-bot or
   out-of-range slot; `Lobby::fill_vacant_with_bots` fills every empty seat and is a no-op when none are
   empty.
2. `crates/core`: a scripted match where player A's bomb eliminates player B — assert `B.eliminated_order
   == Some(1)` and `A.kills == 1`; a self-elimination scenario — assert the eliminated player's own
   `kills` is unchanged and no other player's `kills` increments.
3. `crates/core`: `bombs_placed` increments exactly once per successful placement, independent of whether
   the bomb later detonates, is defused, or is kicked.
4. `crates/net`: round-trip (de)serialize the changed `AddBot{slot}`/`RemoveBot{slot}` v2 requests and the
   additive `kills`/`bombs_placed`/`eliminated_order` snapshot fields; a payload with a legacy `gender` key
   under `Look` still decodes (value discarded) instead of erroring.

## Browser smoke scenarios (manual or CDP-driven, per existing 006/007 smoke patterns)

1. **Menu → lobby**: launch the client; confirm exactly three main-menu actions; "Join a game" opens a
   room list with Refresh and New-room controls; "Create new game" opens the existing create-room form.
2. **Bot fill + per-slot controls**: create a 4-capacity room alone; confirm the other 3 seats show bots
   immediately. As owner, click a bot's cross — confirm that seat goes empty and shows "+"; click "+" —
   confirm a bot fills it again. Join as a second (non-owner) browser session — confirm no cross is shown
   on any seat for that session, and the "+" on any remaining empty seat is still usable.
3. **Configuration panel**: as owner, open the configuration icon; confirm name/mode/size/time/capacity/
   friendly-fire/password are editable there and nowhere else in the lobby body; save and confirm the
   change reflects in the lobby header; confirm a non-owner session never sees the icon.
4. **Leave + action row**: confirm a leave icon sits beside the configuration icon; confirm the lobby's
   bottom action area shows only "Ready" (all sessions) and "Start" (owner session only); confirm "Add
   bot," "Remove bot," and "Change look" buttons are gone.
5. **Options tabs**: open Options from the main menu — confirm Customize/Sound/Controls tabs; open Sound
   and Controls from the in-game pause menu — confirm each lands on the matching tab of the same screen.
6. **Customize**: open the Customize tab — confirm no gender/"Look" selector; confirm Head/Face/Costume
   selectors exist; click the small randomize icon on the preview's bottom-right corner — confirm color,
   head, face, and costume all change; confirm there is no separate full-size "Random" button.
7. **Focus outline**: tab through the main menu, lobby, and options screens — confirm no red outline
   appears on any focused element, and some non-red focus indicator is still visible.
8. **Scoreboard**: play a 4-player match (bots) to completion with a known elimination order; confirm the
   scoreboard's first row is the winner/last-survivor, remaining rows are in reverse elimination order,
   and each row shows only name, kills, and bombs placed. Repeat for a mode/outcome with a draw to confirm
   a stable, deterministic order still results.

## Gates before merge (per constitution)

- `cargo fmt --check`
- `cargo clippy --all-targets -- -D warnings`
- `cargo test --workspace`
- Existing JS smoke suite under `crates/web` passes
- README/WEB docs updated for the new menu/lobby/options/scoreboard layout and the `AddBot`/`RemoveBot`
  slot parameter
