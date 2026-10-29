# Room Protocol Contract

**Status**: Design for implementation, not currently available.

## Transport and Negotiation

Protocol 2 retains tagged JSON objects (`type`) on newline-delimited TCP and one object per WebSocket text message. UTF-8 frame/line limit stays 65,536 bytes; binary, malformed and oversized messages fail safely under existing abuse handling. Never split one object across protocol messages. A TCP connection is initially unjoined; change `Client::connect` callers so transport creation does not implicitly admit a player. Keep a compatibility helper only if it explicitly selects/creates a room; no hidden first-room choice.

Client sends `Hello { version: 2 }`; host replies `Hello { version: 2, server_id, server_epoch, capabilities: ["rooms", "resume", "directory"], limits, defaults }`. `defaults` contains only the public existing room settings for creation; clients missing that optional field use the established defaults. Advertise only capabilities actually implemented at an intermediate checkpoint. Timeout is two seconds. Version-1 `Join` gets legacy-shaped `Rejected { reason: "BadVersion" }` then closure. Other unsupported handshakes get `Error { code: "BadVersion", message }`. A v2 client encountering an old host's unsupported-message response or timeout reports upgrade-required and never falls back to `Join`.

All control requests after Hello include `request_id`, a client-generated printable ASCII identifier of 1-64 characters. Replies echo it; unsolicited state broadcasts do not. At most one admission request is pending per connection. Clients MUST NOT reuse a request ID for a different operation. A duplicate pending admission returns its pending status, never another mutation; a completed duplicate replays its recorded outcome while retained. A client timeout means outcome unknown, not failure or success: reconcile the original operation before submitting a fresh admission. All requests are validated on the authoritative host; no client-supplied actor or owner assertion is trusted.

## Admission Deadlines and Reconciliation

CreateRoom, JoinRoom and the initial ResumeRoom commit have a five-second host-monotonic deadline starting at request acceptance, including worker queue time. At or after that deadline, or after cancellation, a disconnected requester or room closure, pending admission MUST NOT commit; late worker completion is discarded. The coordinator serializes cancellation against commit: cancellation before commit yields Cancelled, whereas cancellation after commit reports the committed outcome and does not undo membership. An explicit LeaveRoom is required to leave an already committed admission.

RequestStatus and CancelRequest identify the original operation through `target_request_id` on the same connection. Completed admission outcomes, including the private original reply, are retained for 60 seconds in a bounded eight-entry cache per connection; reserve a cache entry before accepting work and return Busy rather than evict an unexpired result. These records are never public or logged. RequestStatus returns pending, committed, rejected, cancelled, expired or unknown, plus the original private result when retained. Unknown means no retained result, not permission to rerun an operation; if reconciliation is no longer possible, disconnect and return to explicit recovery/selection rather than silently submit another admission. Disconnect cancels pending work; it does not roll back an already committed membership. Cross-connection resume replay uses the separately authenticated rotation rule below, not this connection-scoped cache.

## Client Requests

| Type | Fields besides type/request_id | Preconditions and outcome |
| --- | --- | --- |
| `ListRooms` | `scope: local|known`, optional `cursor` | No membership required; local never fetches peers; known reads bounded cached aggregation |
| `CreateRoom` | `settings`, `capacity`, optional `password`, `name`, optional `look` | Outside room; valid policy and server capacity; creates room and admits creator atomically |
| `JoinRoom` | `server_epoch`, `room_id`, `name`, optional `look`, optional `password` | Outside room, lobby, free seat, verified password; sent only to destination host |
| `ResumeRoom` | `server_epoch`, `room_id`, `membership_id`, `proof` | Outside room, matching unexpired reservation; same-operation replay is the explicit exception to outside-room checks; password not required |
| `AckResume` | `room_id`, `resume_request_id`, replacement `proof` | Bound membership confirms receipt of its new proof; idempotent, no further rotation |
| `RequestStatus` | `target_request_id` | Query own connection-scoped admission outcome; never rerun the mutation |
| `CancelRequest` | `target_request_id` | Cancel own pending admission or report its already committed outcome |
| `SetRoom` | `room_id`, `expected_revision`, `settings`, `capacity`, optional `password` | Owner, lobby; absent password keeps verifier, empty removes it; atomic full validation |
| `RemovePlayer` | `room_id`, `membership_id` | Owner only; target must be another human member; bot takeover if active |
| `LeaveRoom` | `room_id` | Own current membership; revoke proof, remain connected outside room |
| `Start`, `AddBot`, `RemoveBot` | `room_id` | Owner; existing mode/phase rules, respects capacity and reservations |
| `SetReady`, `SetLook` | `room_id`, existing payload | Own current membership, lobby; look validation unchanged |
| `Input` | `room_id`, existing tick/movement/bomb/punch | Bound connected member of running room; stale ticks validated per room |
| Developer commands | `room_id`, existing payload | Developer mode plus room-owner authority; never all-room mutation |
| `Ping` | existing `nonce` | Outside or inside room, preserves current liveness behavior |

Never trust `room_id` to choose another room on behalf of a bound connection. `room_id` must match binding before actions dispatch. Slot IDs alone cannot address a player across rooms. Fresh admissions never replace a reserved human or silently claim a takeover bot.

## Host Replies and Broadcasts

| Type | Required content |
| --- | --- |
| `Rooms` | `request_id`, `sources`, `rooms`, optional opaque `next_cursor`; source freshness/status and host reference accompany summaries |
| `Welcome` | `request_id`, room reference, `membership_id`, `slot`, private `resume_proof`, `reconnect_window_seconds:60`, explicit owner ID/slot, room revision/policy, existing config/lobby/looks/dev |
| `Resumed` | Same membership/policy/proof fields as Welcome plus phase, current tick, current observation when running, latest result when applicable; no rewind |
| `ResumeAcknowledged` | `request_id`, room reference, `resume_request_id`; receipt confirmed, previous proof revoked |
| `RequestStatus` | `request_id`, `target_request_id`, status, optional original private result; no public/broadcast delivery |
| `RoomUpdated` | room reference, revision, owner identity, capacity/protection flags, membership connection/reservation status and existing lobby/looks |
| `LeftRoom` | `request_id`, room reference; sender now outside room |
| `Removed` | room reference and safe reason; receiver now outside room and must clear proof |
| `RoomClosed` | room reference and reason: `empty|shutdown`; all proofs revoked |
| `Error` | echoed request ID when known, typed code, bounded user-safe message; never echo passwords/proofs or raw invalid input |
| Existing `Start`, `Snapshot`, `PlayerLeft`, `Ended`, `DevState` | add room reference; retain existing gameplay values and semantics |

Proofs appear only in private Welcome/Resumed results, including their authenticated RequestStatus replay, never broadcast, directory, errors or logs. `RoomUpdated` exposes enough explicit owner data that clients no longer call first-human-slot ownership inference. On resume send a coherent current snapshot after binding; queued input from the retired connection is discarded before restoring human control.

### Acknowledged Proof Rotation

On first successful ResumeRoom, create one replacement proof and retain a private rotation record keyed by room/membership and the original request ID. The previous proof authorizes only an authenticated replay of that exact operation, until the earliest of AckResume, ten seconds after first commit, the original reservation deadline, or explicit revocation. A replay returns the same replacement proof without another rotation, membership, join sequence or extended deadline. Ordinary use of a superseded proof, a different request ID or replay after acknowledgment/expiry is rejected.

The client stores the replacement proof before sending AckResume with that proof and the original resume request ID. A lost Resumed response is retried using the original request ID/previous proof; a lost acknowledgment response is retried with the replacement proof. Repeated acknowledgments succeed idempotently for the same bound membership without restoring an old proof. The replacement proof is valid immediately; using it for a fresh resume also confirms receipt and revokes the previous proof.

A same-operation retry on the current connection does not bind again. A retry on a new connection may rebind only after the prior connection is disconnected; otherwise return Busy. Discard retired-connection input on rebind. Before acknowledgment, transport loss preserves the original reservation deadline rather than granting another 60 seconds; replay never extends the ten-second window. At the replay cutoff revoke only the previous proof; the replacement remains valid under the normal membership rules. Leave, removal, reservation expiry and closure revoke both proofs and the private replay record immediately. Persist the original request ID with the private client recovery state until acknowledgment so a page reload can retry the same operation.

## Errors and Atomicity

Codes: `BadVersion`, `BadName`, `BadRoom`, `BadPassword`, `Full`, `InMatch`, `RoomNotFound`, `RoomClosed`, `AlreadyInRoom`, `NotInRoom`, `NotOwner`, `LookAdjusted`, `StaleRevision`, `CapacityConflict`, `ModeNeedsPlayers`, `ResumeExpired`, `ResumeDenied`, `RequestExpired`, `Cancelled`, `Busy`, `RateLimited`, `ServerFull`, `Unsupported`, `Unavailable`.

`LookAdjusted` preserves the existing private, non-fatal colour-collision notice: admission or a lobby look change succeeds with a distinct colour, and only that player is told the assigned public colour. It never indicates a failed admission or changes another participant's look.

Unknown reservation identities and incorrect proofs both use `ResumeDenied` without confirming a private identity. A known expired proof may use `ResumeExpired`. Wrong/missing protected-room password uses the same `BadPassword`. Invalid requests do not create/change membership. Coordinator commits happen only after worker completion is checked against current room revision, password revision, phase, reservation expiry, request deadline/cancellation, live requester binding and free capacity; stale completions fail rather than admit against an obsolete verifier. Two simultaneous final-seat commits yield exactly one success.

## Public HTTP Directory

`GET /rooms` returns aggregate cached directory JSON; `GET /rooms?scope=local` returns local-only data. Content-Type is `application/json`, `Cache-Control: no-store`; cursor and source fields follow Rooms. This is read-only and never accepts passwords or admits players. Unsupported scope/cursor receives 400. Each response is <=64 KiB and at most 32 summaries; follow `next_cursor` for more. Maximum four pages per source. Cursors expire after five seconds and bind source/epoch/revision; mutation invalidation yields `StaleRevision`, allowing one fresh restart rather than an unbounded retry loop.

The host provides equivalent bounded local/known data through its handle so HTTP does not connect through a player identity. Keep existing same-origin WebSocket checks. Do not enable wildcard CORS or `--allow-any-origin` for discovery. Operator-configured net workers use TCP local-only queries, not recursive HTTP crawling.

## Cross-Server Joining

Directory entries carry host and room identity, never an authorization grant. A CLI connects to the configured destination game address and repeats Hello/JoinRoom there. A browser navigates to the configured destination web origin using `#server=<id>&epoch=<epoch>&room=<id>` with escaped values; destination cleans the fragment after selecting the room. Only identity, never names/passwords/proofs, goes in that fragment. Destination prompts as needed and applies its own policy.

While browsing or selecting, preserve existing membership. Switching requires explicit leave confirmation; after acknowledged LeaveRoom the client attempts the destination. If that attempt fails, show retry/room selection; no atomic cross-server rollback is promised. HTTPS sources reject downgrade to HTTP. Invalid configured origins/identity mismatch cause unavailable feedback, not arbitrary redirection.

## Operational Bounds

- Default 256 host connections, configurable 4-1024; web connection limit remains enforced separately and cannot exceed host maximum in integrated startup.
- Max 32 directory entries/page, four pages/source, 16 known sources; public counts cannot exceed four participants per room.
- Control allowance: 20 requests/second/connection with burst 40; gameplay Input uses existing tick validation and a 60/second ceiling. At most 128 queued commands/connection; disconnect abusive senders rather than accumulate unbounded work. Drain at most 256 coordinator commands before rechecking due ticks, rotating room iteration order for fairness.
- Password/resume failure allowance: five attempts/minute/source address, aggregate 100/minute/host; apply bounded accounting (4,096 entries with expiry/LRU). In-process clients use connection identity. Preserve remote peer address through the web bridge, trusting forwarded headers only under explicit configured proxy trust; otherwise rate-limit by socket peer.
- Password workers: two; queue 32. Admission deadline five seconds, completed-result retention 60 seconds, eight reserved result slots/connection. Resume replay window ten seconds, capped by the original reservation deadline and acknowledgment. Exhaustion returns Busy, not blocking simulation.
- Discovery workers: four, one queued/in-flight poll per source, fair oldest-due scheduling; two seconds bounds the complete source attempt, including connection/Hello, pagination and one cursor restart, not each page independently. Distinguish the five-second healthy-directory availability target from failed-source detection: the latter budget is five seconds until due plus queue wait plus two seconds for the attempt. With N <=16 configured sources and four workers, queue wait is at most `(ceil(N/4)-1)*2` seconds, yielding seven seconds for the two-server guide and at most 13 seconds at the configured maximum. Fake-clock tests assert these scheduling bounds; healthy cached/local reads never wait for those attempts.
- Transport writes use bounded per-connection queues; drop slow clients and trigger reservation instead of retaining infinite snapshots.
- All limits are implementation acceptance tests, including malformed JSON, unknown enums, oversized Unicode input and delayed completion after room closure.