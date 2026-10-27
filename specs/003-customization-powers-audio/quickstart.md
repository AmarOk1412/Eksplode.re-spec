# Quickstart & Validation Guide: Customization, Powers, Sound, Developer Mode

All commands run from `game/`. Stages follow the plan; run the gates of a stage before the next one.

## Common gates

```sh
cargo fmt --check
cargo clippy --workspace --all-targets -- -D warnings
cargo test --workspace
(cd crates/web/js-tests && node --test)
./scripts/build-wasm.sh          # wasm <= 300 KB, committed artifact refreshed
python3 scripts/pty-enter-check.py
scripts/check-scenarios.sh       # now also checks the 003 scenario ids
```

## S1 - Customization

```sh
cargo test -p eksplode-re-net --test looks          # lenient decoding, colour resolution, SetLook phase rule
cargo test -p eksplode-re-web --test looks          # looks over WebSocket, same in Lobby for every client
cargo test -p eksplode-re-web-render                # skins per look
(cd crates/web/js-tests && node --test)           # editor state, storage, palette consistency
cargo run -p eksplode-re-web -- --bots 2
```

Open `http://127.0.0.1:8080/`: change colour, gender and accessories and watch the preview update (US1 AS1); reload and see the same look (AS2); join with a second tab choosing the same colour and read the adjusted-colour notice (AS4); start and check both characters in the match (AS5); change the look in the lobby and see it on the other tab (AS6); join from `eksplode-re-cli join 127.0.0.1` and see a default look (AS7). `node scripts/browser-smoke.mjs http://127.0.0.1:8080/ /tmp/s1` saves screenshots.

## S2 - Powers and maluses

```sh
cargo test -p eksplode-re-core            # pickups, maluses, punch, drops, determinism
cargo test -p eksplode-re-bots            # danger from bombs in flight, bonus and malus scoring, soak
cargo test -p eksplode-re-cli             # punch keys, letters, status line
```

Play a round with `--seed 42 --bots 3`: pick up a roller and a flame (US2 AS2); with the dev drop table (S5) or a prepared test arena, check the punch hops over blocks and players and lands on the first free square (AS3, AS4), slow and hyper speed change speed for 10 seconds (AS6, AS7), and small bomb limits reach to one square (AS8). Terminal players see letters and the effect list (AS9).

## S3 - Semi-3D lobby

```sh
cargo test -p eksplode-re-web-render --test iso3d
```

Open the lobby with 4 participants and compare the cards with the reference image: tall cards, large cubic character with a face, name banner, player name under the card, empty cards for free slots, idle bob (US1 AS3).

## S4 - Noise, music, settings

```sh
(cd crates/web/js-tests && node --test)   # fake audio context tests
```

In the browser press a key, hear the lobby tune, start a match (match tune), listen for place, explosion, bonus, malus, punch, landing, elimination and result sounds (US3 AS1, AS2). Open the speaker button: move the music and effects sliders, mute, reload (AS3, AS4). Open the page with sound blocked or in a private window (AS5) and switch tabs (AS6). Record results in `checklists/manual-audio.md`.

## S5 - Developer mode

```sh
cargo test -p eksplode-re-net --test dev
cargo test -p eksplode-re-web --test dev
cargo run -p eksplode-re-web -- --dev --bots 3 --seed 7      # prints the dev warning
cargo run -p eksplode-re-web -- --bots 3                     # normal server: no panel, commands refused
```

As owner, open the dev panel: pause bots (they stop within 1 s, bombs keep burning), set `Punch` to 100% and the rest to 0%, destroy blocks and see only punch icons, reset to defaults (US4 AS1-AS5). Start without `--dev` and send a dev command with the smoke script's raw mode: expect `Error{DevDisabled}` (AS5).

## Final

```sh
cargo test -p eksplode-re-web --test mixed sc003 -- --nocapture   # p95 latency still <= 200 ms (SC-010)
python3 scripts/pty-enter-check.py --attempts 20
node scripts/browser-smoke.mjs http://127.0.0.1:8080/ /tmp/final
```

Follow the README quick start once on a clean clone; docs (README controls, `docs/GAMEPLAY.md`, `docs/AUDIO.md`, `docs/PROTOCOL.md`, `docs/WEB.md`) must pass the docs tests.
