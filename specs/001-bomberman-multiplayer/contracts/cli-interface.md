# Contract: Command-Line Interface

> **Updated for the real behaviour** (feature 002, task T084). The planned native window
> (`eksplode_re-gui`) was **replaced by the web page** of `specs/002-web-client`; there is no GUI
> binary. The options of the web server and of `host --web-port` are in
> `specs/002-web-client/contracts/cli-and-server-options.md`.

Binary: `eksplode-re-cli` (terminal). The browser version is served by `eksplode-re-server`.

```text
eksplode-re-cli [OPTIONS]
eksplode-re-cli host [--port 7878] [--bots N] [--name NAME] [--seed N] [--web-port N] [--bind IP]
eksplode-re-cli join <ADDR[:PORT]> [--name NAME]

Options:
  --players <N>     number of local human players, 0-4 (default 1)
  --bots <N>        number of bots, 0-4 (default 4 - players: fill up to 4 participants)
  --seed <U64>      RNG seed (default: from system time)
  --lobby           choose the slots on a lobby screen before the game starts
  --help, --version
```

Rules:
- `players + bots` must be between 1 and 4 (0 is allowed with `--lobby`), otherwise exit code 2
  with a message on stderr.
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
