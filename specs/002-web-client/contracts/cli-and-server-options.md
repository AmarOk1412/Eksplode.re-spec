# Contract: Server and Command-Line Options

## `eksplode-re-server` (new binary, crate `eksplode-re-web`)

```text
eksplode-re-server [OPTIONS]

Options:
  --bind <IP>             address to listen on (default 127.0.0.1)
  --http-port <N>         web page and WebSocket port (default 8080)
  --game-port <N>         terminal client port (default 7878)
  --bots <N>              bots to add to the lobby, 0-3 (default 0)
  --seed <U64>            fixed RNG seed for every match (default: random)
  --allow-origin <ORIGIN> accept WebSocket connections from this web origin (repeatable)
  --allow-any-origin      accept WebSocket connections from any web origin (unsafe)
  --max-connections <N>   open HTTP/WebSocket connections (default 64)
  -h, --help, -V, --version
```

Behaviour:
- Starts the authoritative match host on `--game-port` and the web server on `--http-port`, both on `--bind`.
- Prints (stdout), for example:
  ```text
  Eksplode.re server running
    Web page:        http://127.0.0.1:8080/
    Terminal client: eksplode-re-cli join 127.0.0.1:7878
    Stop:            Ctrl+C
  ```
- Warnings (stderr), each on its own line starting with `WARNING:`:
  - bind is not loopback: no encryption, no authentication, anyone who can reach the ports can play; use `--bind 127.0.0.1` or a firewall/tunnel to restrict;
  - `--allow-origin`: cross-origin pages from `<ORIGIN>` can drive the game (CORS relaxed);
  - `--allow-any-origin`: any web page opened by anyone who can reach this server can drive the game (unsafe).
- Exit codes: 0 after a clean stop (Ctrl+C), 1 runtime error (for example port in use, with the port and reason in the message), 2 usage error.
- Ctrl+C: sends `HostLeft` to every client, closes sockets and exits with 0.

## `eksplode-re-cli` changes

```text
eksplode-re-cli join <ADDR[:PORT]> [--name NAME]    existing; see below for changes
eksplode-re-cli host [--port N] [--bots N] [--name NAME] [--seed N] [--web-port N]
```

- **`host --web-port N`** (new): additionally starts the web page on port `N` (same origin rules and warnings as the server; bound to `127.0.0.1` unless `--bind` is given) so browser players can join the game hosted from the terminal. Without the option nothing changes.
- **`host --bind <IP>`** (new): listening address (default: the game port on `0.0.0.0` as before, the web page on `127.0.0.1`; an explicit `--bind` applies to both). A non-loopback address prints the `WARNING:` exposure line to stderr before the terminal switches screens (so the default `host` prints it, because its game port listens on every interface).
- **`join`** (changed behaviour):
  - the address may be a web address (`http://`, `https://`, `ws://`, `wss://`): only the host is kept (the web port is not the game port) and the default game port `7878` is added;
  - the lobby accepts Enter (including LF, Ctrl+J, Ctrl+M) to start when the user is the lobby owner, and shows `Only <name> can start the match` otherwise;
  - `b`/`x` add/remove a bot for the owner; all refusals are shown in the notice line;
  - connection failures are explained (refused, full, wrong version) and exit with status 1.
- **Lobby keys everywhere**: Enter start, `b` add bot, `x` remove bot, `r` ready, `q` quit; the keys work the same for the host and for any joined owner.
- Exit codes unchanged (0 ok, 1 runtime error, 2 usage error).

## Documentation contract
Every option above appears in `docs/WEB.md` or the README with an example; a test fails when one is missing (see data-model, documentation set).
