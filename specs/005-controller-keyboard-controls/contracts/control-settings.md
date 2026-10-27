# Control Settings Contract

## Web Controls Panel

- A controls button opens a labeled panel from the web game without leaving the current room or match.
- The keyboard section shows the active key(s) for move up/down/left/right, bomb and punch. The defaults retain arrows plus WASD, Space plus E, and Z plus Q.
- Selecting a keyboard action enters capture mode. One available key replaces that action's aliases. A key already assigned elsewhere or reserved by browser/game navigation is rejected with a visible message; the prior mapping remains active.
- A reset action restores the defaults. Valid web bindings are stored locally and loaded on later visits; invalid or unavailable storage falls back safely to defaults.
- The controller section reports whether a compatible controller is visible, lets the player choose one when multiple are connected, and displays the default Xbox assignments: left stick or D-pad to move, A to bomb, B to punch.
- A controller not yet visible prompts the player to focus the page and press a controller button or move a stick. Missing, blocked or unsupported controller access leaves keyboard play available.
- Controller selection is temporary for the current page. Disconnecting or changing selection clears held controller input.
- Controller movement uses a 0.25 stick dead zone. A held A/B button triggers one action on press, not one action per input tick.
- For simultaneous input, controller movement takes precedence only while it is non-neutral; keyboard movement is the fallback. Bomb and punch presses from either source are accepted once on the next tick.

## Terminal Keyboard Configuration

- Local CLI play accepts independent `--p1-keys`, `--p2-keys`, `--p3-keys` and `--p4-keys` values. Each value is six comma-separated bindings in `up,down,left,right,bomb,punch` order; omitted slots retain their current defaults. A network-joining CLI player can supply the equivalent six-binding value with `--keys` for their own controls.
- Bindings are single characters or the names `Up`, `Down`, `Left`, `Right` and `Space`; character bindings are case-insensitive. Duplicate keys across actions/slots and reserved quit (`q`), pause (`p`) and rematch (`r`) keys are rejected as usage errors.
- Existing mappings remain unchanged when no custom option is supplied.
- These options configure the current CLI run and are not written to a persistent file. A joining player cannot change another participant's controls.

## Game Input Boundary

- The browser continues to send the existing input message fields for movement, bomb and punch on the existing input cadence.
- The terminal continues to submit existing player actions to the local session.
- No controller or key-binding preference is sent to the server. No network protocol or game-rule contract changes.