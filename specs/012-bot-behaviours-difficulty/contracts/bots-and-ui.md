# Contract: Bots API and UI Surfaces

## `eksplode_re_bots` public API

```text
pub enum Behaviour { Aggressive, Protective, Balanced }
  const ALL: [Behaviour; 3]
  fn pick(seed: u64) -> Behaviour            // uniform, deterministic
  fn name(self) -> &'static str              // "aggressive" | "protective" | "balanced"
  fn from_name(&str) -> Option<Behaviour>
  fn label(self) -> &'static str

pub struct Skill { think_every: u32, react_ticks: u32, mistake_chance: f32 }
  fn of(d: BotDifficulty) -> Skill

pub struct Profile { behaviour: Behaviour, difficulty: BotDifficulty }
  fn pick(seed: u64, d: BotDifficulty) -> Profile
  impl Default -> Balanced / Hard

impl DefaultBot
  fn new(seed: u64) -> DefaultBot            // unchanged meaning: Balanced / Hard
  fn with_profile(seed: u64, p: Profile) -> DefaultBot
  fn profile(&self) -> Profile
impl Bot for DefaultBot                       // decide(&Observation) -> PlayerInput, unchanged signature
```

Guarantees:

- `DefaultBot::new(s)` produces the same inputs as before this feature for any observation sequence.
- `decide` reads only the `Observation` (no hidden state of other players).
- Same seed + profile + observations → same inputs.
- No input ever moves into a square burning at the time of the decision (all difficulties).

## Web client (`crates/web/assets`)

| Screen | Control | Owner | Others |
| --- | --- | --- | --- |
| Create room form | "Bot difficulty" select: Easy / Medium / Hard, preselected Medium (or last saved) | editable | n/a |
| Lobby configuration panel | same select bound to the room's value | editable in lobby | read-only (disabled) |
| Lobby header summary | `… - Medium bots` | visible | visible |
| Developer panel (`#dev-panel`, developer mode only) | "Bot behaviour" select: Random / Aggressive / Protective / Balanced, bound to `DevState.behaviour` | editable | read-only (disabled) |

`room.js`: `defaultRoom()` includes `bot_difficulty: "medium"`; `sanitizeRoom()` maps invalid values
to `"medium"`; `roomReducer` handles `{type: "bot_difficulty", value}`.

## Terminal client (`crates/cli`)

| Place | Control |
| --- | --- |
| Host setup screen | "Bot difficulty" row, `< Medium >`, Left/Right cycles |
| Online lobby (owner) | `d` cycles Easy → Medium → Hard; others get "Only the room owner…" notice |
| Lobby settings line | shows `- Medium bots` via `RoomSettings::label()` |
| Usage text | documents `--bot-difficulty` (host only), the `d` key, and the `--dev` bot-behaviour option |
