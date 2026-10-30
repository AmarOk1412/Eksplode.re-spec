# Phase 0 Research: Menu, Lobby, Customization and Scoreboard Reorganization

## 1. How lobby ownership and actions work today

**Decision**: Keep the existing implicit-ownership model — `Lobby::owner()` returns the lowest slot held
by a human (`Controller::Local | Controller::Remote`); there is no separate "owner" flag to migrate.
Every new owner-only UI affordance (kick, remove-bot, configuration panel, Start) is gated the same way
`roomControls()` already gates today's inline room-settings controls (`crates/web/assets/room.js:122-138`),
and the host re-validates ownership server-side for every v2 request exactly as it already does for
`SetRoom`/`Start`.

**Rationale**: This is the only ownership model in the codebase (`crates/core/src/lobby.rs:161-166`); a
new explicit-owner field would duplicate state that can go stale and would be a scope increase the spec
does not ask for. Ownership transfer when the owner disconnects is pre-existing behavior (spec Edge
Cases) and is unaffected.

**Alternatives considered**: An explicit, persisted "owner" identity (e.g., tied to a reconnect token)
was considered so ownership survives a slot reshuffle, but no such reshuffle is introduced by this
feature — slots don't move — so the simpler existing model is kept.

## 2. Mapping lobby actions to the existing v2 protocol

**Decision**: Reuse the v2 `Request` variants already defined in `crates/net/src/protocol.rs` for every
lobby action except bot removal, which needs a slot parameter:

| Spec action | Existing v2 message | Change needed |
| --- | --- | --- |
| Create room | `CreateRoom` | none |
| List/refresh rooms | `ListRooms` | none |
| Join room | `JoinRoom` | none |
| Kick a player (owner cross) | `RemovePlayer { room_id, membership_id }` | none — already slot/membership targeted |
| Remove a specific bot (owner cross) | `RemoveBot { room_id }` | **today frees "the highest bot slot" only** (`Lobby::last_bot`); needs a `slot: u8` field so a specific bot seat's cross removes *that* bot |
| Add a bot to an empty seat ("+") | `AddBot { room_id }` | today fills "the first free slot" (`Lobby::add_bot`), which already matches "click + on an empty seat" as long as the UI only shows "+" on the first free seat scanning order used by the server, or the server is given a `slot` too for precision |
| Open/save room configuration (name/mode/size/time/capacity/friendly-fire/password) | `SetRoom { room_id, expected_revision, settings, capacity, password }` | none — already bundles everything FR-011 needs, including password |
| Ready | `SetReady { room_id, ready }` | none |
| Start | `Start { room_id }` | none |
| Leave room | `LeaveRoom { room_id }` | none |

**Rationale**: Minimizes protocol churn; `RemovePlayer` already takes a `membership_id`, i.e. it is
already precise enough for the "kick a specific player" cross. `AddBot`/`RemoveBot` are the only two
messages that are currently slot-imprecise by design (`Lobby::add_bot`/`Lobby::last_bot` pick a slot
*for* the caller); giving the UI's per-slot "+/×" controls a `slot: u8` on both matches user intent
exactly ("remove *this* bot," "fill *this* seat") and removes ambiguity if, e.g., two seats are empty and
the UI's "+" on seat 3 should not silently fill seat 1 instead.

**Alternatives considered**: Leaving `AddBot`/`RemoveBot` slot-less and relying on the server's
first-free/last-bot picks was considered, since in the steady state (fill left-to-right) it usually
produces the same visible result. Rejected because it breaks the moment seats are non-contiguous (e.g.,
seat 1 human, seat 2 empty, seat 3 bot — clicking "+" on seat 2 must not be allowed to instead do nothing
or affect seat 4), which the acceptance scenarios in User Story 2 require to be seat-specific.

## 3. Bot-fill-on-vacancy policy

**Decision**: Add a small helper, `Lobby::fill_vacant_with_bots(&mut self)` (pre-start only), called by
the room host (`crates/net/src/room.rs` / `host.rs`) at: (a) room creation, immediately after the creator's
own slot is assigned, and (b) whenever a slot transitions to `Controller::Empty` while the room is still
in `Playing == false` / lobby phase (a player leaves and is not replaced by the existing bot-takeover
path, or the owner explicitly removes a bot/player and the resulting empty seat is not left empty by an
explicit "removal" action per FR-008). Mid-match slot vacancies continue to use the existing bot-takeover
behavior untouched.

**Rationale**: FR-008 requires this to be automatic and seat-filling, with the single carve-out that an
owner's explicit removal should show a "+" rather than instantly refilling itself (otherwise "remove this
bot" would be indistinguishable from a no-op). The distinction is time-based, not state-based: fill
happens on room creation and on *other* vacancy causes (e.g. a human leaving before start), but an
owner-initiated `RemoveBot`/`RemovePlayer` leaves that one seat empty until a player joins or a "+" is
clicked.

**Alternatives considered**: Auto-refilling even after an explicit removal (so "remove bot" always
re-adds a different bot immediately) was rejected — it would make the cross control appear broken from
the player's point of view, contradicting User Story 2's explicit "+" affordance on an empty seat.

## 4. Kills, bombs placed, and elimination order

**Decision**: Add three fields to `core::state::Player` (`crates/core/src/state.rs`): `kills: u32`,
`bombs_placed: u32`, `eliminated_order: Option<u32>` (a 1-based ordinal, `None` while alive or for a
winner/survivor). `bombs_placed` increments at the existing bomb-placement site (`crates/core/src/bomb.rs`,
where a new `Bomb` is pushed for a player's input). `eliminated_order` is stamped (next unused ordinal)
at the two existing elimination sites: `crates/core/src/step.rs:185` (blast) and
`crates/core/src/closing.rs:125` (closing-wall/"block lands"). `kills` is incremented for every *other*
player whose bomb contributed to the flame/effect that eliminated a victim, using the existing
`Flame.owners` bitmask (`crates/core/src/state.rs:189-195`) at the same elimination sites; a player's own
bomb eliminating only themselves does not increment anyone's `kills`.

**Rationale**: These are the minimum facts needed to satisfy FR-017/FR-018/FR-019 and are derivable
exactly, deterministically, and in `core` (Principle III/IV) from state the simulation already computes
at those two call sites — no new rule, no new RNG, no new cross-tick bookkeeping beyond incrementing
counters and stamping an ordinal.

**Alternatives considered**: Deriving kills/order after the fact in the web client from the event/log
stream was considered, but the project's event log is for operator diagnostics (per 008's event work),
not guaranteed complete/stable client-side state to build a scoreboard from, and would violate
Core/Frontend Separation by pushing a scoring rule's interpretation into JS. Rejected in favor of
first-class `core` fields.

**Scoreboard ordering derived from this data** (FR-017): sort by `eliminated_order` descending among
players who have a value (most-recently eliminated first), with every player whose `eliminated_order` is
`None` (i.e., alive at match end — the winner, or all survivors in a draw) sorted before all eliminated
players; tie-break equal/absent cases by `kills` descending, then by original slot index ascending, to
guarantee a deterministic order (spec Acceptance Scenario 4 under User Story 5).

## 5. Removing the gender "Look" selector while keeping rendering variety

**Decision**: `gender`/`Gender` stays as an internal-only rendering detail in
`crates/web-render/src/look.rs` / `iso3d.rs` (it drives silhouette pixels at `iso3d.rs:207`), but is
removed from the *player-facing* customization surface: `crates/net/src/look.rs`'s wire-level `Look`
drops `gender` as a settable field (old clients' `gender` in `SetLook`/`Join.look` is read-and-discarded
for backward compatibility, matching the existing lenient-decode pattern noted in `look.rs`'s doc
comment), and the editor UI (`crates/web/assets/look.js`, `index.html`) removes the `#gender` select
entirely. A new `Costume` category (`crates/net/src/look.rs`, `crates/web-render/src/look.rs`) is added
as a genuinely new, player-chosen cosmetic option alongside Head and Face, rendered in
`crates/web-render/src/iso3d.rs` the same way Head/Face overlays already are. Internally, each slot is
still assigned a gender-silhouette the way `defaultLook()` already alternates today
(`crates/web/assets/look.js:27-30`), just no longer exposed or randomizable from the UI.

**Rationale**: The user's complaint ("Female/Male is useless") is about the *control*, not necessarily
about discarding the silhouette variety the renderer already has; keeping it as a fixed, non-selectable
rendering detail satisfies FR-004 (no gender selector) without a riskier full rewrite of the character
sprite system in this feature. Costume is specified as "an implementation/content decision" in the spec's
Assumptions, so its concrete option list (e.g., outfits) is a content choice made during implementation,
not a planning blocker.

**Alternatives considered**: Fully removing the Gender type and unifying the body silhouette into one
shape was considered (simpler model, no hidden internal field) but would be a visible, uncalled-for
visual regression — the sprite work in `iso3d.rs` already renders two distinct silhouettes well; this
feature's scope is the UI control, not sprite redesign. Rejected for this feature; could be revisited in a
dedicated visual-design feature later.

## 6. Scope of the dual-interface parity exception (CLI)

**Decision**: Documented as a scoped exception in the plan's Constitution Check rather than building
new CLI screens. The CLI/terminal frontend (`crates/cli`) has no lobby-seat grid, no options tabs, and no
tabular post-match scoreboard today (results are reported as plain text); none of the six user stories
describe a CLI-side interaction. `core`/`net` data additions (kills, bombs placed, elimination order,
slot-targeted bot removal) are added at the shared layer so a CLI enhancement remains possible without
re-deriving this data, but authoring CLI UI is out of this feature's scope.

**Rationale**: Principle VI requires gameplay features to ship in both interfaces; this is a UI chrome
reorganization of screens that only exist in the web client today, not a new gameplay feature being
withheld from the CLI. No existing CLI screen is being left stale or inconsistent with new game rules —
there are no new game rules.
