# Research: Game Modes and Room Settings

All items of the Technical Context are resolved; there is no open NEEDS CLARIFICATION. Numbers marked *tuning* can change without changing a rule.

## R1 - What a "room" is in this version

- **Decision**: A room is the existing lobby plus a `RoomSettings` value. There is one room per server. The room is *open* while at least one person (browser or terminal) is in the lobby or the match; when the last person leaves the lobby it *closes* and its settings are forgotten (the host keeps its start-up defaults). `RoomSettings` is a plain, self-contained value (`name`, `mode`, `size`, `minutes`, `friendly_fire`) with its own JSON form, so a later feature can list it on the network without touching the rules.
- **Rationale**: FR-001..FR-005. The current host already has exactly one lobby and an owner (lowest human slot); making the settings part of the lobby needs no new concept and keeps the future "share rooms" work out of this feature.
- **Alternatives**: A `Room` entity with an id and a registry (premature, the next specification will decide ids, discovery and passwords); settings in `HostConfig` only (cannot be changed by the owner and cannot be shown to others).

## R2 - How the page knows to show the creation menu

- **Decision**: New optional client message `RoomQuery` (allowed before `Join`) answered by `RoomInfo { open, settings, humans }`. The page sends it as soon as the socket opens: `open = false` shows the creation menu, `open = true` goes to the normal join (name, look) and then the lobby. `Join` gains an optional `room` (settings); the host uses it only when it creates the room (nobody is in it), otherwise it ignores it. A terminal client that joins an empty room sends no `room` and gets the host's defaults.
- **Rationale**: Keeps every existing message sequence unchanged (an unasked `RoomInfo` would be the first message of every connection and would break the existing tests and clients that expect `Welcome` first), is additive in protocol version 1, and old clients never send or need it.
- **Alternatives**: Sending `RoomInfo` to every new connection (changes the first message); a `CreateRoom` message (two ways to join, more states); deciding in the page from the first `Lobby` (too late, the person has already joined and become owner).

## R3 - Settings on the wire and in `Lobby`

- **Decision**: `Lobby` gets `settings: RoomSettings` (`#[serde(default)]`), so `Welcome` and `Lobby` already carry it to everybody. The owner changes it with `SetRoom { settings }` (lobby only, owner only; `NotOwner`, `WrongPhase`, `BadRoom` for invalid values). Decoding is lenient like looks: a missing or invalid field becomes its default; a settings value that is not an object gives the defaults; the name is trimmed, limited to 24 characters and stripped of control characters, with a default of "<owner>'s room" when empty. Allowed values: `mode` Normal/Team/Chaser, `size` Small/Medium/Large, `minutes` 2/3/5/10 (anything else becomes 3), `friendly_fire` bool.
- **Rationale**: FR-009 and FR-029; the same pattern as `look.rs` is already tested and fuzzed. The host rebuilds the match `Config` from the settings at every start, so a rematch needs nothing else.
- **Alternatives**: Strict decoding (one bad field would reject a whole menu); separate messages per setting (more variants, no atomic change).

## R4 - Mapping settings to the game configuration

- **Decision**: `RoomSettings::to_config(&base)` returns the match `Config`: `size` sets `width`/`height` (Small keeps the base size, which is 15 x 13 by default; Medium 19 x 15; Large 25 x 19), `minutes` sets `round_limit_ticks` (3 keeps the base value, which is 3 minutes by default), `mode` and `friendly_fire` set the new config fields, and Chaser mode sets its own arena tuning. "Default keeps the base" lets tests and tools that start a host with a custom base config (short rounds, tiny arenas) keep working.
- **Rationale**: One place decides what a setting means; `Config` stays the only thing `step` reads, so core remains pure and the observation already carries what drawing needs (size, limit, mode).
- **Alternatives**: Passing settings to `step` (a second source of truth); fixed constants in the host (not testable in `core`).

## R5 - Board sizes and fairness

- **Decision**: Small 15 x 13 (minimum, default), Medium 19 x 15, Large 25 x 19; all odd so the pillar pattern, the corner start areas and the center work unchanged. `Arena::generate` already takes the size from the config; its clear start areas are checked by a test on every size (corners and their two neighbours free, every non-wall square reachable from every start, also in the Chaser layout). The page already fits any size up to 64 x 64; the terminal view already reports "terminal too small" with the size needed (large needs 52 x 26 characters).
- **Rationale**: FR-006; no new generator for Normal and Team.
- **Alternatives**: Free width and height (the owner could create unfair or unplayable boards); more sizes (later).

## R6 - The end pattern

- **Decision**: A new `closing` module owns it. In Normal and Team mode, at tick `T0 = round_limit_ticks - 60 * tick_hz` (so 1200 ticks before the end; `closing_ticks` in the config), the pattern starts. Its order is a clockwise spiral over all inner squares (outer ring first from the top-left, then the next ring in, and so on). Every `interval` ticks the next square whose tile is not a wall is **announced** (`BlockWarned`, a `falling` entry with a countdown of `closing_lead_ticks` = 20, one second); when the countdown ends the block **lands** (`BlockLanded`): the tile becomes a wall, any player on it is eliminated, a landed bomb or a power-up on it is destroyed (no explosion; the owner's active bomb count is released), flames on it are removed; flying bombs are untouched and later land only on floor. `interval = max(1, closing_ticks / N)` where `N` is the number of non-wall inner squares at `T0` (143 on the small board gives 8 ticks, 0.4 s; the large board gives 3 ticks), so the whole board is covered within `closing_ticks + closing_lead_ticks` after `T0`. Squares that are already walls at their turn are skipped without using an interval; destroyed blocks that were skipped earlier cannot be missed because blocks are not walls and are covered like floor. The round never ends by time in these modes: `outcome` ends it when at most one player (team) is alive, with a draw when none is left (the old time draw only stays for the Chaser mode, where it means "the runners survived").
- **Rationale**: FR-011..FR-015 and SC-004: the pace is derived, so every size ends within about 61 seconds after the start of the pattern; everything is counted in ticks and the order comes from the arena alone (determinism); landing as an ordinary step phase fits the existing "eliminate" logic.
- **Alternatives**: A random order (harder to read for players and bots); a shrinking rectangle (does not announce squares one by one); a fixed rate whatever the size (the large board would not finish).
- **Compatibility note**: Existing tests that assume a draw at the time limit (`the_round_always_ends_within_the_limit` and similar) change to the new rule in the same change (the rule is in the spec, FR-014).

## R7 - Team mode

- **Decision**: `Participant.team: Option<u8>`, set when the round is built: slots 0 and 1 are team 0, slots 2 and 3 team 1 (Team mode only; `None` otherwise). `Outcome` gains `TeamWin(u8)`; `outcome` in Team mode: all living players in one team means that team wins, none alive means draw. Friendly fire needs to know who lit a flame, so `Flame` gains `owner` (the owner of the bomb that exploded; chain reactions use the owner of the bomb that made the flame). In `eliminate`, a flame on a player removes them unless `!friendly_fire` and the flame's owner is a teammate and not the player themselves. Bots read `team` from the observation, never hunt teammates and still treat every flame as dangerous.
- **Rationale**: FR-016..FR-019. The owner on the flame is the smallest change that makes the rule exact (chain reactions, several flames on one square).
- **Alternatives**: Dropping flames of teammates at creation time (loses chain-reaction ownership and cannot stay exact); a global no-friendly-fire rule (the spec makes it a setting).

## R8 - Chaser mode

- **Decision**: `Role { Normal, Chaser, Runner }` on `Participant` plus `hearts` and `shield_left`. At the build of a Chaser round the state's seeded RNG picks the chaser among the occupied slots (after the arena, so the arena for a seed is the same whatever the draw). The chaser starts at the center (`w/2, h/2`; the center and the four squares around it are cleared) with `bombs_max = chaser_bombs` (5) and `hearts = chaser_hearts` (3); runners start in their corner squares with `bombs_max = 0`, which the existing placement rule already refuses. The chaser's own corner stays empty. `eliminate` for the chaser: if `shield_left == 0` and a flame is on the square, `hearts -= 1`, `shield_left = shield_ticks` (40, two seconds; *tuning*) and `HeartLost`; at zero hearts the chaser is eliminated. Runners die on the first flame. Results: `ChaserWins` when no runner is alive; `RunnersWin` when the chaser is eliminated or when `round_limit_ticks` is reached with the chaser and a runner alive. No end pattern.
- **Rationale**: FR-020..FR-027; reuses placement, flames, kick and punch; the chaser is the only role that needs new state.
- **Alternatives**: Choosing the chaser by slot (predictable, unfair to the owner) or by the owner (a possible later setting); hearts as shields on every player (not asked).

## R9 - The Chaser arena and random bonuses

- **Decision**: `Arena::generate` takes the layout from the config: Chaser keeps the border and only the pillars with `(x/2 + y/2) % 3 == 0` (about a third), uses a small block density (0.12, *tuning*) and clears the center cross and the corner start areas. Blocks do not hide power-ups in this mode; instead the state keeps `bonus_in` (ticks until the next appearance, `bonus_every_ticks` = 100, *tuning*) and, when it reaches zero and fewer than `bonus_cap` (6) power-ups lie on the board, draws a random free square (no wall, block, bomb, player, power-up) and a kind from a weighted list (speed 3, kick 2, punch 2, slow 1, hyper speed 1, small bomb 1; no extra bomb, no flame) with the seeded RNG (`PowerUpSpawned`). The developer drop table applies to Normal and Team only.
- **Rationale**: FR-022, FR-023: the chosen bonuses give runners ways to fight back (punch and kick send the chaser's bombs back) without ever giving them bombs.
- **Alternatives**: Keeping hidden items under blocks (too few blocks); letting extra bomb appear (runners would get bombs, against the spec).

## R10 - Drawing, bots and audio

- **Decision**: The observation gains `falling: Vec<Falling { pos, ticks_left }>` and the participant fields above; nothing secret is added. `web-render` draws announced squares (a pulsing red frame with a dark block descending as the countdown runs out), landed blocks as walls, team marks (a coloured ring or banner under the sprite and on the card), the chaser mark (a crown-like badge) and hearts above the chaser (blinking during protection). The terminal draws `!!` for an announced square, team letters `A`/`B` in the status line and hearts as `<3 x2`. Bots: the danger map treats an announced square as lethal from its landing tick on; pathing never ends a move on one; the hunt goal skips teammates; runner bots keep away from the chaser (a flee goal when the chaser is within 5 squares) and collect bonuses; the chaser bot is the current bot with runners as enemies. The audio gets `warning`, `impact` and `heart` sounds (generated, same documentation rule as `docs/AUDIO.md`).
- **Rationale**: FR-015, FR-019, FR-026, FR-027; all of it uses only the public observation.
- **Alternatives**: A separate bot per role (the current bot already has the needed parts).

## R11 - The creation menu, lobby and terminal

- **Decision**: Page: a `#room-panel` between the join screen and the lobby (name field, mode cards with a one-line description, size and time selectors, friendly-fire toggle shown for Team, "Create room"); in the lobby a settings summary for everybody and the same controls for the owner (each change sends `SetRoom`); the page may remember the last settings in `localStorage` as menu defaults. Terminal: `eksplode-re-cli host` gets `--room-name`, `--mode`, `--size`, `--time`, `--friendly-fire`; `eksplode-re-server` gets the same as start-up defaults of the creation menu; a setup screen (Up/Down select, Left/Right change, Enter confirm) appears before the lobby of `host` unless a settings option or `--no-setup` is given; the lobby shows the settings and the owner changes them with keys; local play gets `--mode`, `--size`, `--time`, `--friendly-fire`.
- **Rationale**: FR-001, FR-003, FR-010; one `RoomSettings` value in all three places.
- **Alternatives**: Terminal options only (the spec asks for a setup screen too).

## R12 - Compatibility and tests

- **Decision**: All additions use `#[serde(default)]`; unknown `ServerMsg` variants are already skipped by the terminal reader and ignored by the page `reduce`; `Outcome` variants for Team and Chaser can only occur in modes an old client cannot start. Tests: core scenario tests per rule and per scenario id `f4_usN_asM_*`, a determinism test per mode and size, arena fairness on every size and layout, net tests for decoding hostile settings and for owner and phase authority, bots scenario tests plus a 200-round Chaser soak, cli view and key tests, web-render command tests, node tests for the menu and HUD models, fuzz-style tests for the new messages, an extended `check-scenarios.sh`, a smoke screenshot per mode.
- **Rationale**: The constitution (II) and the pattern of features 002 and 003.
- **Alternatives**: Bumping the protocol version (breaks existing clients for no gain).
