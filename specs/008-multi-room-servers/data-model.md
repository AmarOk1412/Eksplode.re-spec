# Data Model: Multi-Room Servers and Discovery

**Spec**: [spec.md](spec.md) | **Protocol**: [contracts/room-protocol.md](contracts/room-protocol.md)

## Server Registry

- `server_id`: operator-configured ASCII identifier, 1-64 characters `[A-Za-z0-9._-]`; generated process-local default if absent. Known hosts require explicit stable IDs.
- `server_epoch`: random 128-bit launch identity; distinguishes old references after restart.
- `rooms`: map of random 128-bit `room_id` to `RoomRuntime`, never derived from room name.
- `connections`: connection ID to negotiated version, rate counters and optional membership binding.
- `known_servers`: at most 16 configured `{server_id, label, game_addr, web_origin}` records; unique IDs, no credentials or URL paths/query/fragment in origin.
- `limits`: max rooms (default 32, range 1-128), max connections (default 256), and fixed queue/rate bounds from contracts.
- `directory_revision`: advances when a public local listing changes; private secrets never participate in listing serialization.
- `admission_operations`: connection-scoped request ID, five-second monotonic deadline, pending/committed/rejected/cancelled/expired outcome and private original reply; at most eight reserved result slots per connection, terminal results retained for 60 seconds. Reserve capacity before accepting work; cancellation/disconnect invalidates pending work, not committed membership. See the protocol's reconciliation rules.

Room references are `(server_id, server_epoch, room_id)`; identical names are legal. A restart changes epoch and invalidates references/proofs. Randomness failure rejects startup or creation safely; it never falls back to simulation seeds or timestamps.

## Room Runtime

- Identity/reference above; `revision: u64` incremented on settings, phase, ownership or occupancy mutation.
- `settings: core::RoomSettings`: existing name, mode, board size, minutes and friendly fire.
- `admission`: capacity, optional private salted Argon2id verifier, public password-required boolean.
- `owner_membership_id`: explicit optional owner; reserved owner can remain absent. Never inferred from the current `Controller` variant or lowest human slot.
- `memberships`: stable human records by membership ID plus slot bindings; bot slots remain represented in the existing lobby.
- `lobby`, `looks`, ready state and per-room developer overrides.
- Optional `running`: game state, seed, tick, pending inputs, bot controllers, results and next due tick.
- `phase`: `lobby | running | closing`; clients retain their results view after return to lobby until dismissal, independently of server phase.

Names: 1-24 printable characters after trim (empty creation name defaults to creator's name plus room suffix, truncated to 24); control characters rejected. Existing mode/board/time options stay unchanged. Capacity is 1-4; Team and Chaser require capacity four and four participants at start. Occupied bot slots and reserved human slots count toward capacity once each; a temporary takeover bot and its human reservation are the same occupied slot. Full rooms do not silently evict bots on fresh join; owners use existing remove-bot action first.

Settings updates validate the complete proposed configuration. Capacity cannot drop below occupied slots. Password is 1-128 Unicode scalar values, at most 512 UTF-8 bytes, with control characters rejected; empty password on explicit update removes protection. Password changes affect fresh joins, not existing memberships or legitimate reserved returns. Active matches reject all settings changes. A password-only revision still invalidates outstanding password checks.

## Human Membership

- Random `membership_id`, stable within the room; `slot` in 0-3, existing name/look and `join_sequence`.
- `status`: `connected | reserved`; `connection_id` only while connected.
- `rejoin_proof`: private 32 random bytes, wire-encoded as 64 lowercase hex characters, wrapped to prevent debug serialization.
- Optional `pending_rotation`: original resume request ID, previous/replacement private proofs, original reservation deadline, first-commit time and acknowledgment state. Same-operation replay lasts at most ten seconds and never beyond the original reservation deadline; AckResume or explicit revocation invalidates the previous proof immediately.
- `reserved_until`: monotonic deadline only while reserved; exactly 60 seconds after detected disconnection, not the last received input.
- Current role/team/hearts/alive state remains in the running simulation, not reconstructed from membership metadata.

On transport loss/silence timeout, mark reserved and use the existing bot controller for that slot during running play; lobby ready becomes false. A valid initial resume before the deadline binds one connection, prepares one replacement proof, sends current state and restores human control. Retain the previous proof only for the bounded same-operation replay defined in the protocol, not a fresh admission. The client stores the replacement before acknowledging; lost replies/acknowledgments replay without a second rotation or controller. Before acknowledgment, another connection loss preserves the original reservation deadline. Failed attempts and same-operation retries never extend expiry. At `now >= reserved_until`, revoke both proofs/replay state and release lobby seat or retain an ordinary bot in a running match. An old proof cannot reclaim that ordinary bot.

Explicit leave/removal revokes immediately, with active-match bot replacement while another human/reservation keeps the room alive. No persistent ban list: a removed person may freshly join an eligible lobby with correct password; they cannot reclaim the removed identity.

## Ownership

Creator owns the room. Absence preserves ownership for the reservation window. Explicit leave, removal/expiry of owner membership transfers to the connected human with smallest join sequence, then slot as tie-breaker. If none are connected but valid reservations remain, owner is vacant; the earliest eligible returning human becomes owner. A returning former owner does not displace a transferred owner. Bots never administer rooms.

## Room Lifecycle

```text
outside -- CreateRoom(valid) --> lobby (creator + startup-default bots)
lobby -- Start(owner, valid participants) --> running
running -- match ends --> lobby (same settings and human memberships)
lobby/running -- last human explicitly leaves, no reservations --> closing
lobby/running -- last reservation expires, no connected humans --> closing
closing -- cleanup and RoomClosed --> removed from directory
```

Closing revokes credentials, clears pending inputs, shuts down room-specific work, publishes removal and unbinds members. Requests and delayed worker results referencing a removed room/old epoch fail safely. Joining/creating while already a member returns `AlreadyInRoom`; clients must explicitly leave first. Host shutdown closes all rooms and invalidates the epoch.

## Discovery Cache

- Per configured source: expected ID, observed epoch/version/capabilities, last successful check, last attempted check, status (`fresh | stale | unavailable | incompatible`), bounded public summaries.
- Poll every five seconds, timeout two seconds, concurrency four. Local directory requests never recursively fetch peers.
- The timeout covers the whole attempt, including handshake/pages/cursor restart. Keep at most one poll queued/in flight per source and schedule oldest due sources fairly. Failure detection is bounded by poll interval plus queue wait plus attempt timeout: seven seconds with the guide's one peer and at most 13 seconds with 16 sources. Healthy/local cached directory availability retains its separate five-second target; reads do not wait for a failed peer.
- At failed refresh, mark last summaries stale immediately; remove stale summaries after 15 seconds without success, retaining source status for explanation. Freshness fields use ages, not shared clock comparisons.
- Local summary fields: reference, name, mode, size, minutes, capacity, occupied, connected humans, reserved humans, bot count, phase, password-required, revision and host endpoint metadata from trusted configuration.
- Peer-advertised origins/addresses never override configured host destinations. Unexpected server identity, unsupported version, invalid counts or oversized pages invalidate that source result.

## Event Record

See [operations contract](contracts/operations-and-navigation.md) for schema and category names. Sequence and epoch identify ordering; room revision and game tick correlate mutations. Optional actor is membership ID, not user-supplied raw data. Only explicitly allowed event details serialize; no `Debug` dumps of messages or runtime state.

## Core Boundary and Test Seams

Net policy never adds passwords, clocks, room IDs or network permissions to pure core settings. Each room invokes the unchanged simulation with its own inputs and seed. Service clock, random-source interface, worker completion, directory transport and event sink are injectable so race/expiry/backpressure tests are deterministic without real 60-second waits.