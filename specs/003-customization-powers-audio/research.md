# Research: Customization, Powers, Sound and Developer Mode

All unknowns of the Technical Context are resolved here. No `NEEDS CLARIFICATION` remains.

## R1 - Where the look lives and how it travels

- **Decision**: A `Look` type in a new `net::look` module. `Join` gets an optional `look` field, a new `SetLook { look }` message changes it in the lobby, and the host publishes `looks` (four entries, one per slot) inside `Welcome` and `Lobby`.
- **Rationale**: The look never affects a rule, so it stays out of `core` (principle III). The lobby message is already broadcast on every change, so everybody sees updates for free.
- **Alternatives**: Put the look in `core::Lobby` (mixes cosmetics into rules and the bots' observation); a separate `Looks` message (two messages to keep consistent).

## R2 - Palette, gender looks, accessories, defaults

- **Decision**: 10 named colours (`Blue, Black, Red, Purple, Green, Orange, Yellow, Pink, Teal, White`), gender `Male | Female`, head accessory `None | Cap | TopHat | Crown | Headphones`, face accessory `None | Glasses | Sunglasses | Moustache | Mask`. Slot defaults: slot 0 Blue/Male, 1 Black/Female, 2 Red/Male, 3 Purple/Female, no accessories (the four existing characters). Wire values are the names as strings.
- **Rationale**: Meets FR-001 (at least 8 colours, two accessory places with at least four choices plus none). Names on the wire survive reordering and are readable in logs.
- **Alternatives**: Free RGB colour (no distinct-colour guarantee, harder to resolve conflicts, accessibility issues); numeric ids (fragile).

## R3 - Colour conflicts

- **Decision**: On `Join`/`SetLook`, if another seated participant (human or bot) has the requested colour, the host picks the next free colour in palette order after it (wrapping) and sends the sender `Error{code:"LookAdjusted", message}`. Slots that were seated earlier keep their colour. Bots and terminal players get the slot default, resolved the same way. Leaving frees the colour.
- **Rationale**: Deterministic, local to the host, gives the "nearest free colour and told" of FR-005 without a new message type.
- **Alternatives**: Reject the colour (annoying); allow duplicates (violates FR-005); let clients negotiate (race conditions).

## R4 - Lenient decoding of a hostile or old look

- **Decision**: `look` is accepted as loose JSON (`Option<serde_json::Value>`); `Look::from_wire` takes each field, and any missing or unknown value becomes the slot default for that field. A non-object value yields the full default. Nothing in a look can produce a `Malformed` error.
- **Rationale**: FR-007 and FR-030: never break other players; a strict enum parse would reject the whole `Join`.
- **Alternatives**: Strict serde enums (an old page or typo gets `Malformed`, which blocks joining).

## R5 - Persistence on the device

- **Decision**: `localStorage` keys `eksplode.re.look.v1` and `eksplode.re.audio.v1` holding small JSON objects; reading is wrapped so corrupt or unavailable storage falls back to defaults. A new version bumps the suffix.
- **Rationale**: No accounts (spec assumption); privacy mode and blocked storage must not break the page.
- **Alternatives**: Cookies (sent with every request); server-side profiles (needs identity).

## R6 - Look rendering in the match

- **Decision**: `web-render` builds a `Skin` from the look: body colour = chosen colour; helmet = the colour mixed with white; fixed neutral face and eye colours; Black keeps its orange-eye variant by rule (dark body gets orange eyes and buckle). Gender adds pixel overlays (female: ponytail pixels at the sides and lashes; male: none). Accessories are pixel overlays on named rows (cap and hat above the helmet row, crown spikes, headphone cups at the sides, glasses and moustache on the face rows). Templates become layers over the existing 12x13 template. JS passes the four looks once per lobby/match start through `eksplode_re_set_looks` (JSON), not per frame.
- **Rationale**: One source of drawing truth (principle III spirit), cheap per frame, and the four players stay distinct because colours are unique (R3).
- **Alternatives**: Pre-baked sprite sheets (many combinations, large); drawing accessories in JS (duplicates art decisions).

## R7 - Decide items at block destruction

- **Decision**: Remove `hidden_powerups`. `GameState` owns a `DropTable` (eight chances in `PowerUpKind` order). For each destroyed block, draw one number from the state RNG; walk the cumulative chances; at most one item. Default chances are the existing overall rate (`powerup_chance`, default 0.3) times the shares of the spec: roller .22, flame .22, extra bomb .19, small bomb .11, slow .10, hyper speed .08, kick .05, punch .03 (so punch is 0.9% of blocks, 3% of drops). If the sum exceeds 1.0 the table is scaled down proportionally.
- **Rationale**: FR-020 and FR-028 (live per-power chance). The observation already hid unrevealed items, so bots and clients see no difference. Same seed and inputs still give the same result (principle IV).
- **Consequence**: Existing tests that assert which block hides what must be rewritten around destruction events; the hidden-item fixture of `web-render` is unaffected (it only holds visible items).
- **Alternatives**: Keep hidden items and re-roll them on a table change (changes already generated blocks, surprising); two RNG streams (needless complexity).

## R8 - Power kinds and effects in `core`

- **Decision**: `PowerUpKind` becomes `ExtraBomb, Range, Speed, Punch, Slow, HyperSpeed, SmallBomb` (the first three keep their names on the wire; "roller" and "flame" are display names). Bonuses (first four) are permanent for the round: `Participant.has_punch`, `bombs_max`, `range`, `move_cooldown`. Maluses use one slot `Participant.malus: Option<ActiveMalus { kind, ticks_left }>`. Effective cooldown at move time: slow `ceil(base * 4 / 3)` (about 25% slower), hyper speed `max(1, base / 2)` (about double). Small bomb sets the range of each placed bomb to 1. Duration `Config.malus_ticks` = 10 s at the tick rate (200). A new malus replaces the old one; picking the same one again restarts the timer; elimination clears it.
- **Rationale**: Matches FR-011..FR-016 and keeps bonuses and maluses independent. Computing at move time avoids drift from repeated multiplication.
- **Alternatives**: Multiply `move_cooldown` in place and restore later (rounding errors, order bugs when bonuses are picked during a malus).

## R9 - Punch mechanics

- **Decision**: A new `PlayerInput.punch` flag (default false). In a tick, after movement and before bomb placement, a player with `has_punch` who is alive and presses punch targets the bomb on the square they face (the neighbour in `facing`, only if that bomb is not already in flight). The bomb gets `flight { dir, hop_left }`; every `Config.punch_hop_ticks` (2 ticks = 100 ms) it advances one square in `dir`. On each arrival it lands if the square has no block, wall, living player, or other bomb (landed or in flight); otherwise it keeps hopping. If it would leave the arena, it lands on the nearest free square going back toward its origin (the origin square counts); only if none is free does it keep hopping, so a punch never destroys a bomb. The fuse keeps burning in flight; a bomb in flight is not triggered by flames and, if its fuse ends, explodes at its current square. Flying bombs do not block movement. Events: `BombPunched`, `BombHopped`, `BombLanded`.
- **Rationale**: Implements "hops square per square until a free square" and defines every edge case in the spec deterministically. Immune-in-the-air keeps chain reactions understandable.
- **Alternatives**: Teleport to the landing square (loses the visible hops); fixed 3-square flight as in classic Bomberman (not what was asked); bombs triggered in the air (hard to reason about).

## R10 - Punch key and terminal parity

- **Decision**: Terminal punch keys: P1 `z`, P2 `c`, P3 `u`, P4 `n` (free in every layout; `q`, `p`, `r`, `b`, `x` keep their meanings). Browser: `KeyZ` or `KeyQ`. The terminal draws powers as letters with colour (`R` roller, `F` flame, `B` extra bomb, `P` punch, `S` slow, `H` hyper, `s` small bomb) and the status line lists active effects with the remaining seconds of a malus.
- **Rationale**: FR-014 and FR-017 in both clients (principle VI).
- **Alternatives**: Re-using the bomb key with a modifier (terminals do not report modifiers reliably).

## R11 - Bots with the new rules

- **Decision**: The danger map treats a bomb in flight as exploding at its predicted landing square (same rule as `core::punch`) and also marks its current square; target selection values bonuses and gives maluses a negative weight; bots never press `punch` in this feature (documented, optional later). All stay deterministic.
- **Rationale**: Principle V, no cheating, no stuck bots; keeps scope bounded.
- **Alternatives**: Smart punch use (a feature of its own).

## R12 - Semi-3D lobby

- **Decision**: `web-render::iso3d` projects tile/cube coordinates with a 2:1 isometric projection and draws, per card, a ground shadow, a body cube, a head cube (three shaded faces) whose front faces carry the pixel face as skewed quads, and accessory boxes (hat, crown spikes, headphone cups, glasses strip). An `eksplode_re_portrait3d(slot, width, height, phase)` export returns commands for the card; `phase` drives a gentle bob and a slow yaw swap between two views. Cards are painted on a `requestAnimationFrame` loop only while the lobby is visible.
- **Rationale**: Reproduces the reference (large cube head, 3/4 view) with the existing command buffer; testable natively (bounds, determinism, per-slot differences).
- **Alternatives**: CSS 3D transforms (a second rendering path, harder to test); WebGL (big new surface); pre-rendered images (no customization).

## R13 - Audio: generated, not recorded

- **Decision**: `audio.js` builds an `AudioContext` on the first user gesture and three gain buses (master, music, effects). Effects are recipes of oscillators with envelopes plus filtered noise bursts: bomb place (short blip), explosion (noise burst with falling low-pass plus a sine thump), bonus pick-up (rising arpeggio), malus pick-up (falling buzz), punch (pitch-drop whack), punched bomb landing (thud), elimination (descending tones), round result (win or draw jingle), button click (tick). Music is a lookahead step sequencer (25 ms timer, 120 ms horizon) playing patterns built from a scale and a fixed seeded generator: lobby 96 BPM major pentatonic with arpeggio and bass, match 144 BPM natural minor with driving bass and hi-hat noise. Snapshot `events` are mapped to sounds on the page.
- **Rationale**: FR-021..FR-025, no copyright issues (no samples, no melodies from other works), reproducible and unit-testable (note lists and gain values can be checked against a fake context).
- **Alternatives**: Audio files (license risk, size); a synth library (new dependency); `<audio>` elements (poor timing and mixing).

## R14 - Settings and browser audio rules

- **Decision**: Settings object `{ music: 0..1, effects: 0..1, muted: bool }` with defaults `0.35 / 0.7 / false`; applied to bus gains with a short ramp (about 50 ms) so changes are heard within 0.2 s; stored under `eksplode.re.audio.v1`. The context starts on the first key or click; music pauses (context suspend) when the page is hidden and resumes when visible; all audio calls are no-ops if `AudioContext` is missing or throws.
- **Rationale**: Satisfies FR-024, FR-025 and SC-008; the game stays fully playable silent.
- **Alternatives**: One volume only (spec needs music and effects separately).

## R15 - Developer mode

- **Decision**: `ServerOptions.dev` / `HostConfig.dev` set by `--dev` on `eksplode-re-server` and `eksplode-re-cli host`. New client messages (all owner-only and refused with `Error{code:"DevDisabled"}` when dev is off, `NotOwner` for non-owners): `DevPauseBots { paused }`, `DevSetDrops { chances: [f32; 8] }`, `DevResetDrops`. The host keeps `bots_paused` and the drop table, applies the table to the running `GameState` between ticks, skips bot decisions while paused (bots get an empty input; bombs keep burning), and broadcasts `DevState { paused, drops }` on change and on join. `Welcome` carries `dev: bool` so the page shows the panel and a "DEV MODE" marker. Chances are clamped to 0..=1 and non-finite values rejected; a sum above 1 is scaled proportionally when rolling. Start-up prints a `WARNING:` that dev mode is for testing and must not be exposed to untrusted networks.
- **Rationale**: FR-026..FR-029 with the host as the only authority; pausing in the host needs no change to bot code.
- **Alternatives**: A separate debug port (more surface); client-side pausing (not authoritative).

## R16 - Compatibility and tests

- **Decision**: All new fields use `#[serde(default)]`; unknown `ServerMsg` variants are skipped by the terminal reader and ignored by the page's `reduce`; `Input` without `punch` means false. Tests: core scenario tests per rule, net tests for decoding hostile looks and dev authority, web tests for looks over WebSocket and dev refusal, a consistency test that the options listed in `look.js` equal those in `net::look`, web-render tests for skins and the 3D portrait, node tests for editor, settings, sound mapping and music scheduling, fuzz-style tests extended to the new messages, and an extended `check-scenarios.sh`.
- **Rationale**: Old clients keep working (additive v1) and every acceptance scenario is covered.
- **Alternatives**: Bumping the protocol version (breaks existing terminal clients for cosmetics).

## R17 - Kick (bomb sliding)

- **Decision**: A new bonus `Kick` appended to `PowerUpKind::ALL` (the drop table grows to eight entries, the existing indexes stay). `Participant.has_kick`; `Bomb.slide`. The action is a move into a landed bomb, so no new input and no protocol change beyond the observation fields and the eight-entry drop table. Slide speed is `slide_ticks` (3). The rule is in `core` next to the punch (`kick.rs`), bots use `predict_slide_stop` for danger.
- **Rationale**: FR-032, FR-033; classic Bomberman behaviour; deterministic, counted in ticks (principle IV); works the same in CLI, page and bots (principle VI).
- **Naming**: requested as "roller", already taken by the speed bonus, so the game calls it kick.
- **Alternatives**: Kick on a key (needless, both clients would need a key); push the bomb one square per step (not what was asked, the bomb should run to the next blocker).

## R18 - Pixel-art icons and directional, animated player sprites

- **Decision**: Icons are small character templates (10x10 for the board) in `sprites.rs`, drawn like the bomb with `runs`; below a small tile size a letter is drawn instead. Player sprites become `grid(look, view, frame)` with views Down, Up and Left (Right mirrors Left) and frames idle, step A, step B. The renderer derives view and frame from data it already has: facing (from the position change or `facing` in the observation) and the interpolated position (distance walked picks the step frame; no movement picks idle). Portraits keep the front idle frame.
- **Rationale**: FR-011, FR-034, FR-035 with no wire change; the style stays flat top-down pixel art.
- **Alternatives**: Sending an animation state from the host (more wire surface, no gain); sprite sheets (a binary asset, against the generated-art approach).
