# Research: Bot Behaviours and Difficulty Levels

All Technical Context unknowns are resolved below. Codebase facts come from
`game/crates/bots/src/default_bot.rs`, `game/crates/core/src/{room.rs,config.rs,state.rs}`,
`game/crates/net/src/{room.rs,host.rs}`, `game/crates/cli/src/{app.rs,setup.rs,online.rs,args.rs}`,
`game/crates/web/src/args.rs` and `game/crates/web/assets/{room.js,app.js,index.html}`.

## R1. Where behaviour and difficulty live

- **Decision**: `BotDifficulty` (Easy/Medium/Hard) is a plain data enum in `core::room`, stored in
  `RoomSettings`. `Behaviour` (Aggressive/Protective/Balanced) and the per-difficulty tuning table
  (`Skill`) live in the `bots` crate, combined in a `Profile { behaviour, difficulty }` passed to
  `DefaultBot::with_profile(seed, profile)`.
- **Rationale**: The difficulty is a room setting that `net`, `cli` and `web` already exchange through
  `RoomSettings` (same path as `friendly_fire`); `core` must stay free of bot logic (Principle III),
  so it only carries the value. Behaviour is never a setting, so it does not leak into `core` or the
  wire.
- **Alternatives considered**: Putting the difficulty in `Config` (sent to every client with `Start`,
  but bots are host-side only and `Config` is rule data, not seat data); a new trait object per
  behaviour (three `Bot` impls duplicating the safety layer, rejected: the safety rules of FR-008 must
  stay shared).

## R2. Behaviour picking (FR-002..FR-004)

- **Decision**: `Behaviour::pick(seed)` draws `Rng::new(seed ^ BEHAVIOUR_SALT).next_below(3)`
  from the bot's existing per-seat seed (`match_seed * 4 + slot` at match start,
  `matches * 8 + slot` on mid-match takeover). The bot's own decision RNG is untouched.
- **Rationale**: Deterministic per seed and seat (FR-003, Principle IV), uniform (FR-002),
  independent per bot, and re-picked on every new match/rematch because the match seed changes.
  Using a separate salted RNG keeps the decision RNG stream of a Balanced/Hard bot identical to
  today's (FR-015).
- **Alternatives considered**: Drawing from the bot's decision RNG (shifts every later tie-break, so
  existing seeded tests would change); picking in `net` from the OS RNG (not reproducible from the
  match seed).

## R3. Behaviour priorities (FR-005..FR-009)

Shared safety layer (unchanged, runs first for every behaviour): punch-away for runners, flee danger /
desperate step, runner flees chaser, closing pattern moves inward, `can_bomb_here` (escape check and
friendly-fire check).

| Step after safety | Balanced (today) | Aggressive | Protective |
| --- | --- | --- | --- |
| 1 | power-up (depth 8) | hunt (depth 20) | power-up (depth 8) |
| 2 | hunt (depth 14) | power-up (depth 4) | dig (depth 14) |
| 3 | dig (depth 14) | dig (depth 14) | hunt (depth 5) |
| 4 | wander | approach nearest enemy (depth 20) | keep away: step to a safe neighbour that maximises distance to the nearest enemy when one is within 3 |
| 5 | — | wander | wander |

- Protective goal predicates also reject target squares adjacent to an enemy (FR-006), and its
  runner flee distance is `FLEE_DISTANCE + 2`; Aggressive chaser keeps today's `pressing` rule.
- As built, a Protective bot also steps back from a threat within 3 squares *before* the
  opportunistic "bomb here" check, so it trades bombs only when cornered or from range; without
  this the behaviours differed too little to meet SC-004 (measured on the same 100 seeds with four
  Aggressive bots versus four Protective bots: 13 335 vs 6 658 hits, mean survival 2 224 vs
  3 317 ticks).
- **Rationale**: Reuses existing `find_path` goals with different order/depths, so every behaviour is
  a small table-driven variation tested with scripted scenarios (Principle V). Balanced is exactly
  today's order (FR-007, FR-015).
- **Alternatives considered**: Utility scoring with weights (harder to test with scripted scenarios
  and to keep Balanced bit-identical); a "risk tolerance" that relaxes the escape check for Aggressive
  (rejected by FR-008).

## R4. Difficulty mechanics (FR-010..FR-014)

Movement takes `base_move_cooldown = 4` ticks per square at 20 Hz; a bomb fuse is 60 ticks.

- **Decision**: Three knobs per difficulty, in the `bots` crate:

  | Knob | Easy | Medium | Hard |
  | --- | --- | --- | --- |
  | `think_every` (ticks between fresh decisions) | 6 (0.30 s) | 3 (0.15 s) | 1 |
  | `react_ticks` (an opponent's bomb younger than this is not yet seen) | 10 (0.50 s) | 5 (0.25 s) | 0 |
  | `mistake_chance` (per fresh decision) | 0.20 | 0.06 | 0.0 |

  - Between fresh decisions the bot holds its last movement direction (never a bomb or punch) and
    drops it if the next square is burning now or not passable (FR-013).
  - A bomb's age is `config.fuse_ticks - ticks_left`, all public in the `Observation`; the bot's own
    bombs are always seen. Bombs older than `react_ticks`, flames and falling blocks are always seen.
  - A mistake, rolled only when `mistake_chance > 0`: if the planned action is a bomb, the bomb is
    skipped (missed opportunity); if the bot is in danger, it hesitates (no input); otherwise it takes
    a wrong turn (a random passable, non-burning neighbour) or hesitates, 50/50.
- **Rationale**: "Slower decision taking" maps to `think_every` + `react_ticks`; reactions stay far
  below the 60-tick fuse so Easy bots usually still escape (spec assumption) but miss quick chain
  reactions and late threats. No RNG draw at Hard keeps Balanced/Hard identical to today. Values are
  starting points: they may be tuned during implementation, keeping Easy > Medium > Hard ordering, to
  meet SC-002/SC-003 in the soak tests. They were kept as is: Hard beats Easy 95/100, Medium
  beats Easy 78/100, Hard beats Medium 81/97 decided rounds; Easy bots eliminated themselves 244
  times against 73 for Hard over 100 four-bot matches.
- **Alternatives considered**: Acting on an observation N ticks old (the bot's own position becomes
  stale, producing nonsense directions rather than plausible slowness); random suicide bombs
  (frustrating, violates FR-008); reduced path depth only (weak effect, not perceived as "slower").

## R5. Room setting and wire compatibility (FR-016..FR-021)

- **Decision**: `RoomSettings.bot_difficulty: BotDifficulty`, serde `#[serde(default)]`, lowercase
  names `"easy" | "medium" | "hard"`, default `Medium`. `net::room::from_wire` reads
  `bot_difficulty`, falling back to Medium on missing/invalid values. Owner-only and lobby-only
  enforcement is inherited from the existing `SetRoom` handling (v2) and legacy `SetRoom` (v1).
- **Rationale**: Same proven path as `friendly_fire`; old clients that omit the field get Medium
  (FR-020, SC-006); no protocol version bump since the field is additive.
- **Alternatives considered**: A separate `SetBotDifficulty` request (duplicates owner/phase checks
  and revision handling already in `SetRoom`).

## R6. Applying the difficulty when bots are created

- **Decision**: Every `DefaultBot::new(...)` call site that creates match bots
  (`net/src/room.rs` start and mid-match takeover, `net/src/host.rs` `bots_for` and takeover,
  `cli/src/app.rs` `make_bots`) becomes
  `DefaultBot::with_profile(seed, Profile::pick(seed, settings.bot_difficulty))`, reading the
  room's settings captured at match start (FR-019). `DefaultBot::new(seed)` stays as
  Balanced/Hard for tests.
- **Rationale**: Bots are host-side only; no client needs to know behaviours.
- **Risk**: Behaviours are random, so integration tests in `net`/`web`/`cli` that depend on exact bot
  moves may change. Such tests either assert outcomes that do not depend on bot moves, or run the
  room in developer mode with the behaviour forced to Balanced (R9) and `bot_difficulty: Hard`,
  which is exactly today's bot (FR-015).

## R7. User interfaces

- **Web**: add a "Bot difficulty" `<select>` (Easy/Medium/Hard) to the create-room form and the
  lobby configuration panel, next to friendly fire; `room.js` `defaultRoom`, `sanitizeRoom`,
  `roomReducer` and the summary line handle `bot_difficulty`; non-owners see it disabled (existing
  `roomControls()` gating).
- **Terminal**: `--bot-difficulty <easy|medium|hard>` for `host` only (`cli/src/args.rs`); a "Bot
  difficulty" row in the host setup screen (`cli/src/setup.rs`); `d` cycles it in the online lobby
  for the owner (`cli/src/online.rs`, key free today). `local` and the headless web server get no
  flag: only the host chooses (FR-021); local bots play at the default Medium.
- **Label**: `RoomSettings::label()` appends ` - <Label> bots` (e.g. `Medium bots`) so both clients
  show it in the lobby line.

## R8. Measuring success criteria

- **Decision**: Extend `crates/bots/tests/soak.rs` with seeded runs: behaviour distribution over 300
  picks (SC-001), 1v1 win rates per difficulty pair over 100 seeds (SC-002), self-elimination counts
  per difficulty (SC-003), bombs reaching an opponent and survival ticks per behaviour (SC-004).
  Scripted scenarios in `crates/bots/tests/behaviour.rs` cover each acceptance scenario.
- **Rationale**: Existing soak harness already runs four-bot matches deterministically.

## R9. Developer-mode behaviour override (FR-023..FR-025)

- **Decision**: A new developer command next to `DevPauseBots`/`DevSetDrops`:
  v2 `Request::DevSetBotBehaviour { request_id, room_id, behaviour: Option<String> }` and legacy
  `ClientMsg::DevSetBotBehaviour { behaviour: Option<String> }`; `null` means Random. The room
  (`net/src/room.rs`) and the legacy host (`net/src/host.rs`, new `DevCommand::Behaviour`) store
  `bot_behaviour: Option<Behaviour>` and use it instead of `Behaviour::pick` whenever bots are
  created (next `Start`, takeovers). `DevState` gains `behaviour: Option<String>` (serde default
  `None`). Same checks as the other dev commands: owner only (`NotOwner`), developer mode only
  (`Unsupported` on the room protocol, `DevDisabled` on the legacy host); an unknown name or type
  is `BadRoom` on the room protocol (like `DevSetDrops` there) and `BadValue` on the legacy host.
  Allowed in the lobby and during a match.
  `Behaviour` gets `name()`/`from_name()` (`"aggressive"`, `"protective"`, `"balanced"`), so the
  `bots` crate needs no serde.
- **Web**: a "Bot behaviour" select (Random/Aggressive/Protective/Balanced) in the existing
  `#dev-panel`, driven by `devPanel(state)` and `DevState.behaviour`.
- **Rationale**: Gives testers and integration tests a deterministic, today-identical bot (Balanced +
  Hard) without exposing behaviour choice to normal players; reuses the dev-mode gating and
  broadcast path.
- **Alternatives considered**: A per-slot override (more UI for no test need); a hidden room setting
  (would leak into normal rooms and `RoomSettings`); a server flag (not changeable at runtime).
