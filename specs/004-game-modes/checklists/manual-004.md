# Manual checks: game modes and room settings (feature 004)

Automated checks live in `cargo test` and `node --test`; scenarios that need eyes or ears are recorded here with the scenario id (`f4_usN_asM`), the date and the result. A line counts for `scripts/check-scenarios.sh` only when ticked.

## US1 - Room creation and settings

- [ ] f4_us1_as1 the room creation menu is the first screen on an empty server and works on a phone-sized window
- [x] f4_us1_as2 a second tab goes straight to the lobby and sees the room name, mode and settings - headless Chromium smoke run (`scripts/browser-smoke.mjs`): a second participant over a WebSocket gets `RoomInfo` open, the same settings in `Welcome`, and `NotOwner` for `SetRoom`
- [x] f4_us1_as3 the owner's change reaches the other tab and the terminal client at once - the smoke run changes the size in the lobby and the peer receives the `Lobby` with 25 x 19 at once; the terminal client is covered by `crates/cli/tests/online.rs`
- [x] f4_us1_as6 the three board sizes fit the page and the terminal (and the terminal says when it is too small) - headless Chromium smoke with the large board (`SIZE=large node scripts/browser-smoke.mjs`: 25 x 19 board drawn whole in the page, clock 04:59 for 5 minutes); terminal sizes and the too-small message are covered by `crates/cli/tests/view.rs`; not tried in a real terminal or on a phone

## US2 - End pattern

- [ ] f4_us2_as6 the announced squares, the landed blocks and the time left are visible; the warning and impact sounds are heard - seen: headless Chromium run `CLOSING=1 node scripts/browser-smoke.mjs` (red frames with a growing dark block, landed walls, the banner "The walls close in!" and the clock `CLOSING`); the sounds are covered by node tests with a fake audio context and still need a listener

## US3 - Team mode

- [x] f4_us3_as5 team colours and letters are recognisable and the result message names the winning team - headless Chromium smoke in Team mode (`MODE=team node scripts/browser-smoke.mjs`): cards with blue and orange frames and "team A/B" marks, rings under the characters, HUD rows with `[team A]`; the result text is covered by node and terminal tests; the end of a real Team round was not watched

## US4 - Chaser mode

- [ ] f4_us4_as9 the chaser mark, the hearts, the protection blink and the heart sound are recognisable; the result says "Chaser wins" or "Runners win" - seen in headless Chromium (`MODE=chaser node scripts/browser-smoke.mjs --dev`): open arena, the chaser in the middle with a crown and three hearts, runners with `[runner]` in the HUD; the heart sound, the blink and a finished round were not watched or heard
- [ ] dev panel: the drop sliders say they have no effect in Chaser mode
