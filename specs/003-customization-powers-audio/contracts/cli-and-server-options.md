# Contract: Server and CLI Options, Controls and Routes

Base: `specs/002-web-client/contracts/cli-and-server-options.md` and `http-and-websocket.md`.

## `eksplode-re-server`

```text
--dev       developer mode: owner-only commands to pause bots and set drop chances (testing only)
```

- Off by default. When on, start-up prints (stderr) `WARNING: developer mode is on: ... for testing; do not expose to an untrusted network.` and the banner shows `Developer mode: on`.
- Documented in the options table of `docs/WEB.md` (generated from `USAGE`; the docs test keeps it in sync).

## `eksplode-re-cli host`

```text
eksplode-re-cli host [--port N] [--bots N] [--name NAME] [--seed N] [--web-port N] [--bind IP] [--dev]
```

`--dev` gives the shared match host the same developer rules; the terminal itself has no panel (the page does). Same warning as the server.

## Controls

| Where | Move | Bomb | Punch |
|-------|------|------|-------|
| Browser | arrows or WASD | Space or E | Z or Q |
| Terminal P1 | arrow keys | Space | `z` |
| Terminal P2 | W A S D | E | `c` |
| Terminal P3 | I J K L | O | `u` |
| Terminal P4 | T F G H | Y | `n` |

The punch key only acts when the player has the punch and faces a landed bomb. The README controls table and the in-page hint list the punch key.

## Terminal drawing

Power-ups appear as letters on their tile: `B` extra bomb, `R` roller, `F` flame, `P` punch, `K` kick, `S` slow, `H` hyper speed, `s` small bomb; bonuses in green, maluses in red. A bomb in flight is drawn as `()` on its current square with a trailing `~`. The status line of each player adds the active effects: `punch`, `slow 7s`, `hyper 4s`, `small 9s`.

## Page

- Join screen: the character editor (colour swatches, gender toggle, head and face accessory pickers, live preview) next to the name field; a speaker button opens the settings (sound).
- Lobby: semi-3D character cards (see render contract), a change-look button, owner/you/ready marks.
- Match: status line shows power icons and the remaining malus time; the dev panel (only when `dev`): pause/resume bots, eight drop sliders 0-100%, reset button, and a "DEV MODE" marker.

## HTTP routes (static, embedded)

| Route | Content |
|-------|---------|
| `/` and `/index.html` | page |
| `/app.js`, `/look.js`, `/audio.js` | ES modules (`look.js` and `audio.js` are new) |
| `/style.css` | styles |
| `/eksplode_re_render.wasm` | graphics module |

All carry the existing security headers and CSP (`script-src 'self'`, no inline script). The route list in `docs/WEB.md`, the HTTP contract and `crates/web/tests/http.rs` are updated; the total page size stays at most 500 KB.

## Tests that bind this contract

Option parsing (`--dev`, with and without `--web-port`); start-up warning and banner line; terminal key mapping for the four punch keys without clashes; terminal view letters and status effects; routes and headers for the new modules; page size limit; docs tests (README controls, options table).
