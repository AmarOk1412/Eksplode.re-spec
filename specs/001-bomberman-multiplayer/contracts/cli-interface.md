# Contract: Command-Line Interface

> **Updated for the real behaviour** (feature 002, task T084). The planned native window
> (`eksplode_re-gui`) was **replaced by the web page** of `specs/002-web-client`; there is no GUI
> binary. The options of the web server and of `host --web-port` are in
> `specs/002-web-client/contracts/cli-and-server-options.md`.
>
> **Updated for feature 011's menu/lobby parity**: local play now shows the lobby screen before
> the match starts by default (matching the web client's main menu), instead of starting
> instantly. `--instant` restores the old instant-start behaviour; `--lobby` is still accepted
> (now a no-op) for compatibility with existing scripts/docs.
>
> **Updated (2026-10-29)**: a bare `eksplode-re-cli` (no subcommand) now defaults to `host`
> instead of local play, so a plain run is immediately joinable by others — matching the web
> client's own default of landing somewhere ready for others, not a solo match. Local-only play
> moved under an explicit new `local` subcommand carrying the same options local play always had;
> `host`/`join` are unchanged except that both now also accept `-V`/`--version` (previously only
> `-h`/`--help` worked there, an oversight exposed once `host` became reachable without typing it).

Binary: `eksplode-re-cli` (terminal). The browser version is served by `eksplode-re-server`.

```text
eksplode-re-cli [OPTIONS]                        defaults to host (below)
eksplode-re-cli host [--port 7878] [--bots N] [--name NAME] [--seed N] [--web-port N] [--bind IP]
eksplode-re-cli join <ADDR[:PORT]> [--name NAME]
eksplode-re-cli local [OPTIONS]                  play on this machine only, no network

Local options (`local` subcommand):
  --players <N>     number of local human players, 0-4 (default 1)
  --bots <N>        number of bots, 0-4 (default 4 - players: fill up to 4 participants)
  --seed <U64>      RNG seed (default: from system time)
  --lobby           choose the slots on a lobby screen before the game starts (default)
  --instant         skip the lobby screen and start the match immediately
  --help, --version
```

Rules:
- `players + bots` must be between 1 and 4 (0 is allowed with the lobby screen, i.e. without
  `--instant`), otherwise exit code 2 with a message on stderr.
- `host --bots` accepts 0-3 (the host takes a slot); the default is 0.
- stdout/TTY used for the game UI; errors go to stderr; exit code 0 normal, 1 runtime error
  (including a refused or failed join), 2 usage error.
- A terminal smaller than 40x20 shows "terminal too small (need WxH, have WxH); q quits" and
  waits for a resize or `q`.

## Controls (terminal, local keyboard)

| Player | Move | Bomb |
|--------|------|------|
| P1 | Arrow keys | Space |
| P2 | W A S D | E |
| P3 | I J K L | O |
| P4 | T F G H | Y |
| All | `q` quit, `r` rematch after a result, `p` pause (local only) | |

Lobby keys (online): Enter start (owner only), `b` add bot, `x` remove bot, `r` ready, `q` quit.
In the browser: arrows or WASD to move, Space or E for a bomb.

## Screens
Lobby (slot list, add/remove bot, start) -> Game (arena, per-player status line: bombs, range,
speed, alive) -> Results (winner or draw, rematch/quit).

Glyphs (terminal): wall `##`, block `[]`, floor `  `, bomb `()`, flame `**`, power-ups `+B`,
`+R`, `+S`, players `P1`..`P4` coloured.

## Graphical version (replaced)
The isometric view with coloured cubes lives in the web page (`crates/web-render` compiled to
WebAssembly and `crates/web/assets`). Same rules, same options through `eksplode-re-server`.
