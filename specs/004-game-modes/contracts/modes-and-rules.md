# Contract: Rules of the modes and the end pattern

## Step order (one tick, additions in bold)

1. Read inputs (movement, bomb, punch).
2. Move players (kick, punch as before).
3. Place bombs (a participant with `bombs_max == 0` can never place one; this is how runners work).
4. Advance bombs and detonate (flames record the bomb owner).
5. **Closing: start the pattern at `T0`, announce the next squares, land the blocks whose countdown ended.**
6. **Chaser only: bonus timer and spawn.**
7. Collect power-ups.
8. Eliminate: flames and landed blocks. **Chaser: hearts and shield. Team: friendly-fire filter.**
9. Compute the outcome.

Landing is placed after detonation so a bomb that explodes on the tick a block lands still explodes. Power-ups are collected while a player moves (step 2), so a player who steps onto a power-up in the tick its block lands collects it and is then crushed; a power-up nobody holds is destroyed by the landing.

## Normal

- Rules of feature 003 plus the end pattern from `T0 = round_limit_ticks - closing_ticks`.
- Outcome: last living player wins (`Winner`); nobody alive gives `Draw`. No time draw: the pattern guarantees an end within `closing_ticks + closing_lead_ticks` after `T0` (SC-004).

## End pattern

| Item | Rule |
|------|------|
| Order | clockwise spiral of the inner squares: top row left to right, right column down, bottom row right to left, left column up, then one ring in, repeat |
| Interval | `max(1, closing_ticks / N)`, `N` = non-wall inner squares at `T0` |
| Announce | `Event::BlockWarned`, a `falling` entry with `ticks_left = closing_lead_ticks` |
| Landing | tile becomes `Wall`; player on it eliminated; bomb and power-up on it destroyed without explosion, owner's bomb count released; flame on it removed |
| Skips | squares that are already walls are skipped without using an interval |
| Flying/sliding bombs | unaffected until they stop on a square; a bomb that stops on a falling square is destroyed on landing |
| Blocking | an announced square is still walkable until it lands |
| Start | `Event::PatternStarted` once, so clients can show the warning (sound and banner "The walls close in!") |
| Determinism | pure function of state and tick; no RNG |

A player standing on a square when it lands dies (`PlayerEliminated` in the same tick as `BlockLanded`; the event has no cause field, clients tell a crush from the landing event).

## Team

- Teams: slots 0 and 1 against slots 2 and 3; `Participant.team` set at the start; bots and humans are interchangeable.
- Friendly fire on: every flame hurts everybody. Off: a flame owned by a teammate does not hurt; a flame owned by the player themselves still does; an enemy flame hurts.
- Outcome: `TeamWin(t)` when only team `t` has living players; `Draw` when none is alive. The end pattern applies as in Normal.
- Requires 4 participants (humans and bots; a start with fewer is refused with `CannotStart`).

## Chaser

| Item | Rule |
|------|------|
| Participants | exactly 4; one chaser, three runners |
| Role choice | seeded RNG, after the arena is generated, among occupied slots |
| Start | chaser at the center with 2 bombs and 3 hearts; runners in corners with 0 bombs; arena per research R9 |
| Chaser hit | with no shield: `hearts -= 1`, shield for `shield_ticks`, `HeartLost`; the third heart eliminates; flames during the shield are ignored; chaser's own flame counts |
| Runner hit | eliminated at once |
| Bonuses | every `bonus_every_ticks` while fewer than `bonus_cap` on board: a random free square and weighted kind (speed 3, kick 2, punch 2, slow 1, hyper speed 1, small bomb 1) |
| Bonus effects | the chaser can collect them too (bombs and flame range stay at their start values because extra bomb and flame are not in the list) |
| Outcome | `ChaserWins` when no runner is alive; `RunnersWin` when the chaser is eliminated, or at `round_limit_ticks` with the chaser and at least one runner alive |
| End pattern | none |

## Results table (what the results screen shows)

| Mode | Headline | Per-player details |
|------|----------|--------------------|
| Normal | "<name> wins" / "Draw" | order of elimination |
| Team | "Team A wins" / "Draw" | teams and survivors |
| Chaser | "The chaser wins" / "The runners win" | hearts left, roles |

## Tests

`f4_us2_*`: announcement and landing timing on every size, pattern covers the whole board, crush, bomb and bonus destruction, determinism, round always ends. `f4_us3_*`: teams, friendly fire on/off, self damage, chain reactions, outcome. `f4_us4_*`: roles, hearts and shield, runner death, bonus schedule and weights, arena, results, round limit.
