# Manual checks: customization, powers, sound, developer mode (feature 003)

Automated checks live in `cargo test` and `node --test`; scenarios that need eyes or ears are recorded here with the scenario id (`f3_usN_asM`), the date and the result.

## US1 - Customization and lobby

- [x] f3_us1_as1 the preview changes at once when colour, gender or an accessory changes
- [x] f3_us1_as3 the lobby cards look like the reference image (tall card, large cubic character, name banner, player name below, empty cards) - 2026: headless Chromium capture with four participants (`scripts/browser-smoke.mjs`): rounded tall cards in the slot colours, isometric cubic characters with the pixel face, banner `PLAYER n`, name caption and you/owner/ready marks; black on black reads thanks to a light floor pad; animation checked on two captures 1.3 s apart
- [x] f3_us1_as8 players show the front, back and side view for the direction they move - frames drawn by the wasm module to PNG (idle, step A, step B for four players facing down, up, left and right) and a headless Chromium screenshot of a real match: the back has no face, the side shows one eye and a profile, right mirrors left, the crown and the teal colour stay
- [x] f3_us1_as9 moving players walk (step frames) and standing players idle - the step frames show alternating feet and arms with a one-pixel bob, standing players are in the idle pose (rendered frames); the pace follows the speed by test (`sprites3.rs`), not timed by eye
- [x] f3_us1_as5 every player in a match shows the chosen colour, gender look and accessories and the four can be told apart

## US2 - Powers

- [x] f3_us2_as10 the eight power icons are pixel-art pictures, not letters, and read as bonus or malus at a glance - rendered frame at 3x: bomb with a plus, flame, rollerskate, boxing glove, boot, snail, lightning bolt and tiny bomb with a minus; white frames on dark blue for bonuses, dark red frames on grey for maluses
- [ ] f3_us2_as11 a player with the kick walks into a bomb and it slides to the next blocker - covered by `crates/core/tests/kick.rs` and the render test of a sliding bomb; not played live in a browser or a terminal
- [ ] f3_us2_as12 a sliding bomb is visible as movement and bots keep out of its path - covered by `crates/web-render/tests/icons.rs` and `crates/bots/tests/danger.rs`; not seen live

- [x] f3_us2_as9 power icons are recognisable and bonuses differ from maluses; hops of a punched bomb are visible; malus pulse under the player - checked on a frame drawn by the wasm module (all seven icons, two players with a malus, a landed and a flying bomb) converted to a PNG: icons B F R P with white frames and S H s with dark red frames are told apart by letter and colour, the flying bomb is lifted with a shadow below it, the pulse tint is subtle; the terminal letters and status line are covered by `cargo test -p eksplode-re-cli`; not played live in a terminal by the agent

## US3 - Sound

Done without ears (headless Chromium via `scripts/browser-smoke.mjs`, node tests with a fake audio context): the page loads without console errors, the sound panel opens from the speaker button, a slider and the mute toggle are stored under `eksplode.re.audio.v1`, schedules and recipes match the contract. The lines below need a person with speakers and stay open until someone has listened.

- [ ] f3_us3_as1 the lobby tune and the match tune loop and switch with the stage
- [ ] f3_us3_as2 every sound effect is heard and distinct
- [ ] f3_us3_as3 volume changes are heard within 0.2 s; mute silences everything
- [ ] f3_us3_as6 music pauses in a hidden tab and resumes

## US4 - Developer mode

- [x] f3_us4_as1 the developer panel and the DEV MODE marker appear only on a `--dev` server - headless Chromium via `scripts/browser-smoke.mjs`: `--dev` server shows the panel (sliders set to Punch 100 / rest 0, pause button turns into "Resume bots", console clean); a normal server shows no panel

## Results

(Notes with dates are added below as checks are done.)

- 2026-10-22 S1: headless Chromium run of scripts/browser-smoke.mjs: editor preview updates for Teal / Female / Crown / Glasses (f3_us1_as1); the same character is drawn in the match and the status chip uses the chosen colour (f3_us1_as5); console clean.

## Success criteria (SC-001 to SC-012)

Evidence from the automated gates and the headless checks of this implementation; nothing below was judged by a person looking at a screen for a long time or listening on speakers.

| SC | Result | Evidence |
|----|--------|----------|
| SC-001 | met by design, preview timing not stopwatch-measured | editor preview repaints on every change (`look.test.mjs`, smoke script run) |
| SC-002 | met | `crates/net/tests/looks.rs`, `crates/web/tests/looks.rs`, mixed-match test `f3_us2_as9_powers_work_the_same_for_every_client` compares the looks seen by a browser and a terminal client |
| SC-003 | partly checked by eye | four colours told apart in the lobby and match screenshots; seven icons on one rendered frame |
| SC-004 | met | `crates/core/tests/punch.rs` (free square, blocks, players, bombs, edge, fuse, chain) |
| SC-005 | met | `crates/core/tests/drops.rs` (1000 draws, shares within 3 points, punch at most 5 %) |
| SC-006 | met | `crates/core/tests/drops.rs` (200 draws at 100 % and 0 %) and `crates/net/tests/dev.rs` over the host |
| SC-007 | met | `crates/core/tests/malus.rs` (ends in the 200th tick, values restored) |
| SC-008 | gains and persistence tested, loudness not heard | `audio.test.mjs` (50 ms ramp, mute, reload); by-ear lines of US3 stay open |
| SC-009 | met | `docs/AUDIO.md` and `crates/web/tests/docs.rs` (every id documented as generated, no audio file) |
| SC-010 | latency met, 60 fps by eye not measured | p95 about 100 ms (`sc003`); wasm 193 KB, page 255 KB |
| SC-011 | met | `crates/net/tests/dev.rs` (bots stand still within 20 ticks, `DevDisabled` without `--dev`) |
| SC-013 | met | `crates/core/tests/kick.rs` (about twenty situations: wall, pillar, block, bomb, living and dead player, four no-room cases, no kick, flying bomb, four directions, a player stepping into the path, fuse ending mid-slide, punch on a slide, predictions equal the real stops) |
| SC-014 | met by test and by eye on rendered frames | `crates/web-render/tests/sprites3.rs`, wasm node test, PNG of the four views and three frames |
| SC-012 | not tested with a new reader | `docs/GAMEPLAY.md`, `docs/AUDIO.md`, `docs/WEB.md` and the docs tests |
