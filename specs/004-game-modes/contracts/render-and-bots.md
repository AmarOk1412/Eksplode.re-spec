# Contract: Observation, drawing and bots

## Observation (what drawing and bots may read)

| Field | Use |
|-------|-----|
| `config.mode`, `config.friendly_fire`, `config.round_limit_ticks`, `config.closing_*` | rules and clocks |
| `players[].team`, `.role`, `.hearts`, `.shield_left` | marks, HUD, targets |
| `falling[]` (`pos`, `ticks_left`) | announced squares |
| `flames[].owner` | not needed by clients; kept internal, may be omitted from the wire |
| `arena` | landed blocks are ordinary walls |
| `tick` | pattern time = `tick - (round_limit_ticks - closing_ticks)` |

Nothing hidden is added; the observation stays the only input of bots (constitution V).

## Drawing (`web-render` and terminal)

| Element | Drawing |
|---------|---------|
| Announced square | red pulsing frame, block shadow growing as `ticks_left` falls (page); `!!` (terminal) |
| Landing | block drops in, short dust burst, camera nudge off when reduced motion is set |
| Team | coloured ring under the sprite (team A blue, team B orange), letter on the card |
| Chaser | crown badge above the sprite, hearts row, blinking while `shield_left > 0` |
| Runner | normal sprite, small running icon on the card |
| Closing banner | text banner for 2 seconds at `PatternStarted` |

`web-render` gets new draw commands behind the same raw export interface (no new export function is required: the commands are part of the frame buffer layout; the buffer layout version in `docs/WEB.md` is bumped and tested).

## Bots

| Situation | Behaviour |
|-----------|-----------|
| Falling squares | danger map marks a square lethal from `ticks_left` to the end of the match; a bot standing on or heading into one moves to the nearest safe square; a path never ends on a falling or landed square |
| Team | enemy list excludes teammates; bombs are not placed where only a teammate would be hit unless friendly fire is on and the bot is cornered; never plants next to a teammate when friendly fire is on |
| Chaser (bot) | walks toward the nearest runner, plants when a runner is within flame reach or has a route that ends in a dead end, keeps its escape route, uses the second bomb to cut routes |
| Runner (bot) | flees when the chaser is within 5 squares, prefers wide corridors, picks up bonuses within 6 steps, uses punch and kick on bombs when it has them |
| Difficulty | existing levels keep working; only reaction thresholds differ |

## Tests

Bots: scripted scenario tests for each row (falling-square avoidance with ticks, never plants on teammate, flees chaser, chaser corners a runner) and a soak of 200 rounds per mode that checks no panics, rounds end, and bots survive the pattern more often than a stationary bot. Render: command-buffer tests for each new element. Terminal: view snapshot tests for each new element.
