# Quickstart: Validate Controller and Keyboard Controls

## Prerequisites

- Rust stable and Node.js are installed.
- The repository is checked out on Linux.
- For physical validation, connect the Xbox-compatible controller and open the locally served page in a normal, visible browser window.

## Automated Checks

From `game/`:

```sh
node --test crates/web/js-tests/*.test.mjs
cargo test -p eksplode-re-cli
cargo test -p eksplode-re-web
```

Expected outcomes:

- Browser control tests cover existing keyboard defaults, remapping validation and persistence, controller standard mapping, dead zone, one-shot actions, disconnection and input-source precedence.
- CLI tests cover unchanged defaults, independent custom slot bindings and invalid or conflicting options.
- Existing web and CLI tests continue to pass without changes to the protocol or game rules.

## Linux Controller Check

Start the web game from `game/`:

```sh
cargo run -p eksplode-re-web -- --bots 1
```

Open `http://0.0.0.0:8080/` in a visible browser window, focus the page, open the controls panel and press a button or move the stick on the connected controller. Confirm that the panel reports the controller and its standard mapping. Join a match and verify:

1. D-pad and left stick move in all four directions.
2. Xbox A places a bomb and Xbox B uses punch when available.
3. Holding either action button does not repeat the action every 50 ms.
4. Keyboard input still works; a changed keyboard binding survives a reload and reset restores all original aliases.
5. Disconnecting the controller leaves keyboard input usable and no action held.

If the controller is visible to Linux but absent from the panel, keep the page focused and interact with the controller once. Check whether the browser exposes a `standard` mapping and whether its page policy permits Gamepad API access. Headless browser checks do not replace this physical-device validation.

## CLI Check

Run the CLI help and a local game with the documented custom key options. Confirm omitted player slots keep their defaults, custom movement/bomb/punch bindings affect only the intended slots, and duplicate or reserved keys produce a usage error before play begins. Also join a hosted game from the terminal with a custom key set and verify only that client's inputs change.