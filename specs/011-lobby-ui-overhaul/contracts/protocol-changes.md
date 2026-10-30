# Contract: v2 Room Protocol Changes

Base protocol: `crates/net/src/protocol.rs`, `mod v2`. This document lists only what changes; every
message not listed here (`CreateRoom`, `JoinRoom`, `ListRooms`, `ResumeRoom`, `AckResume`,
`RequestStatus`, `CancelRequest`, `SetRoom`, `RemovePlayer`, `LeaveRoom`, `Start`, `SetReady`, `Ping`,
`DevPauseBots`, `DevSetDrops`, `DevResetDrops`) is reused as-is.

## Changed: `v2::Request::RemoveBot`

```text
Before: RemoveBot { request_id: RequestId, room_id: String }
After:  RemoveBot { request_id: RequestId, room_id: String, slot: Option<u8> }
```

**Implementation note (011 implementation)**: `slot` is `Option<u8>` with `serde(default)` rather than a bare
`u8`, so a request without it still decodes. The web client always sends `slot`; an absent `slot` keeps
the pre-011 "highest bot slot" pick, which the terminal client (`b`/`x` keys) and the v1 `ClientMsg`
translation rely on. Error codes are new typed `v2::ErrorCode` variants: `BadSlot` (index outside the four
seats), `NotABot` (seat holds a human or nobody); a non-owner gets the existing `NotOwner`.

- `slot` is the 0-based seat index the owner clicked the cross on.
- Host validates: sender is the room owner (FR-016); `slot` is in range `0..MAX_PLAYERS`; the slot's
  `Controller` is `Bot`. Any failure returns the existing `Error` response shape with a reason
  (`BadOwner`, `BadSlot`, or `NotABot` — reuse `Error` with a `LobbyError`-derived message, no new
  top-level error enum needed beyond what `v2::Response` already supports for other owner-only actions).
- On success, the slot becomes `Empty` and is **not** auto-refilled (research.md §3) — the UI shows "+".

## Changed: `v2::Request::AddBot`

```text
Before: AddBot { request_id: RequestId, room_id: String }
After:  AddBot { request_id: RequestId, room_id: String, slot: Option<u8> }
```

**Implementation note (011 implementation)**: `slot` is optional for the same backward-compatibility
reason as `RemoveBot` (absent = first free seat). Per FR-010 ("+" usable by any player) and FR-016 (which
does not list adding a bot), `AddBot` is accepted from **any** room member, not only the owner. Errors:
`BadSlot` (index outside the four seats), new `SlotTaken` (seat not empty), existing `Full` (seat past the
room capacity, or the room is at capacity), existing `InMatch`.

- `slot` is the 0-based empty seat the "+" was clicked on.
- Host validates: `slot` in range; the slot's `Controller` is `Empty`. If another client filled the same
  seat first (race), the host returns the existing full/slot-taken error shape, and the requesting
  client's UI reconciles to the latest broadcast lobby state (same pattern as 008's admission races).

## Bot fill (host policy, no message change)

- `CreateRoom`: every empty seat within `capacity` gets a bot right after the creator is seated
  (`--bots`/`default_bots` no longer limits this).
- `JoinRoom`: a joining human takes the first empty seat within capacity, otherwise the **lowest bot
  seat** (the bot is replaced), so a bot-filled room never refuses a human for being "full" of bots.
- `LeaveRoom` / disconnect before the match: the seat gets a bot. Owner `RemovePlayer`/`RemoveBot{slot}`:
  the seat stays `Empty` until "+" or a joiner fills it.
- `SetRoom` capacity: shrinking removes bots (seats past the new capacity first, then highest bots),
  growing fills the newly opened seats with bots. `CapacityConflict` is rejected whenever any **human**
  slot index would fall at or beyond the new capacity — not merely when the human *count* exceeds it.
  Counting humans alone is unsound here: since `resize_bots` never moves a human (only bots are
  stripped), a human sitting at a high slot index can survive a shrink that only checks the aggregate
  count, leaving them outside `admission_slot`'s capacity window — invisible to both the `Full` check and
  the directory's occupancy — so the room can silently admit more humans than its declared capacity.
  (Found by review after initial implementation; fixed with a regression test,
  `set_room_capacity_cannot_strand_a_human_beyond_the_new_window` in
  `crates/net/tests/room_host.rs`.)
- `RoomSummary.occupied` keeps counting bots; clients decide joinability from `connected_humans +
  reserved_humans` (the web room list does).

## Unchanged but newly load-bearing: `v2::Request::SetRoom`

Already carries `settings` (name/mode/size/minutes/friendly_fire), `capacity`, and `password` in one
message — this is the single request the new configuration panel sends on save (FR-011). No shape change.

## Unchanged but newly load-bearing: `v2::Request::RemovePlayer`

Already takes `membership_id`, which identifies a specific human occupant — this is the message the new
per-slot "×" cross sends for a human-occupied seat (FR-009). No shape change. (Bot-occupied seats use
`RemoveBot` above instead, since bots have no `membership_id`.)

## New: per-player match statistics on existing snapshot/result types

Wherever the host already sends per-player state during a match and at match end (the existing
`Observation`/result message carrying each player's status — see `crates/net/src/protocol.rs` and
`crates/core` snapshot types), add three fields per slot:

```text
kills: u32
bombs_placed: u32
eliminated_order: Option<u32>   // null on the wire when None
```

These are additive fields on an existing message; old clients that don't read them are unaffected
(`serde` default-on-missing is not needed on the *sending* side, but the receiving web client must treat
a missing/absent value defensively — e.g. default to 0/`null` — for mixed-version safety during rollout).

## Unaffected: `ClientMsg::AddBot` / `ClientMsg::RemoveBot` (protocol v1, non-room)

The legacy single-room v1 messages (`crates/net/src/protocol.rs` lines ~60-62) are untouched by this
feature; v1 rooms are a separate, already-superseded path per feature 008's research. If v1 is still
reachable, its `RemoveBot` keeps today's "highest bot slot" behavior — only the v2 (multi-room) path gets
slot-targeted bot controls, since the new lobby UI as specified is part of the multi-room client.

## `net::look::Look` / `web-render::look::Look` wire shape

```text
Before: { "colour": Colour, "gender": Gender, "head": Head, "face": Face }
After:  { "colour": Colour, "head": Head, "face": Face, "costume": Costume }
```

- Decoding (`Look::from_wire` equivalent): a `gender` key, if present, is read and discarded (no error) —
  preserves the existing "a bad/old entry only ever gives a default" leniency for the removed field, and
  keeps old stored/cached looks from erroring.
- `costume` absent (e.g. from a client that predates this feature) decodes as `Costume::None` (or
  equivalent default), the same pattern `head`/`face` already use when absent.
