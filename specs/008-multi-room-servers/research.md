# Research: Multi-Room Servers and Discovery

**Date**: 2026-10-28 | **Spec**: [spec.md](spec.md)

## Evidence and Research Questions

Read-only server research examined `game/crates/net/src/{host,client,protocol,codec,room}.rs`, the web bridge, core lobby/settings, manifests and existing tests. Client research examined `game/crates/web/assets/{room,nav,app}.js`, CLI argument/setup paths and the feature 005-007 contracts. The following decisions resolve room ownership, concurrency, discovery, security, compatibility, input preservation and logging; no unresolved design questions remain.

Verified facts:

- `Host::start`, `HostHandle::connect` and `Server::run` in `net/src/host.rs` use standard threads and channels. One server currently owns one lobby and running match. TCP readers decode newline-delimited, tagged JSON enums, not JSON-RPC.
- `protocol.rs` defines version 1, a 64 KiB line limit, `Join`, `RoomQuery`, `SetRoom`, four-slot `Welcome`, and no reconnect or directory messages.
- Core `Lobby::owner()` and browser `roomControls()` infer ownership from the first human slot. That inference cannot represent reserved ownership or join-order transfer.
- `room.js` stores presentation defaults, not admission policy. Passwords and reconnect credentials must not enter this storage or shared core `RoomSettings`.
- The web server already bridges browser WebSockets to in-process/TCP clients. The workspace does not use Tokio, Axum, tracing or an HTTP client library.
- `nav.js`, `controls.js` and `touch.js` already own navigation layers, controller/text entry and mobile pointer behavior. The established input and results contracts remain applicable.

## R1 - Room authority and scheduling

**Decision**: Refactor the net host into a single connection/room registry with per-room `RoomRuntime` state. Keep one public game port, web bridge and existing thread/channel infrastructure. A coordinator orders mutations; each due room advances its own 20 Hz simulation and maintains independent seed, tick, pending input, bots, look, developer state and final result. Process bounded command batches between deadlines; never perform discovery, password work or stdout I/O in this loop.

**Rationale**: The owning abstraction directly controls routing and admission. Explicit room context avoids dynamic port allocation, shared-lobby leakage and implicit creation on `Join`. The scope is modest (default 32 rooms, validation at three full rooms); fair scheduling and bounded queues are sufficient initially.

**Alternatives considered**: Separate host processes/dynamic ports duplicate listeners and complicate browser discovery, permissions and shutdown. An async framework or actor framework would add a runtime without a demonstrated need. Separate simulation workers remain an optimization if load checks fail, not an initial dependency.

## R2 - Membership, reconnect and ownership

**Decision**: Keep `owner_membership_id`, stable membership identity and join sequence in net-owned room state, separate from slot controllers. Use a monotonic injected service clock for 60-second reservations; keep wall time out of core. During absence, a bot controls the same slot. A successful reclaim prepares a 256-bit replacement proof; retain the previous proof only for authenticated same-operation replay until acknowledgment, ten seconds after commit or the original reservation deadline, whichever comes first. Store the replacement before AckResume; lost responses/acknowledgments never create a second controller or extend the original reservation. A bot at an expired active-match slot becomes an ordinary bot. Last-human disconnect preserves the room until the final reservation expires; explicit final departure closes it immediately only when no other valid reservation remains.

**Rationale**: Names are not identity and a low slot is not necessarily the longest-present human. Rejoining must restore the latest state, including elimination, rather than reconstruct a player. Monotonic time makes expiry immune to wall-clock adjustment and fake-clock tests avoid real waits.

**Alternatives considered**: Name-based reclaim enables impersonation; a 64-bit random token has inadequate security margin. Immediate ownership transfer on disconnect contradicts the reserved-ownership policy. Persistent user accounts and restart recovery exceed scope.

## R3 - Credentials and admission work

**Decision**: Add `getrandom` 0.2 for OS randomness, `argon2` 0.5 for salted Argon2id password verification, and `subtle` 2 for constant-time reconnect-proof comparison. Pin compatible releases in implementation and check the workspace MSRV. Keep passwords outside `RoomSettings`; store only salted password verifiers, private reconnect proofs in memory and public `password_required`. Use 19 MiB, two passes, one lane for Argon2id. Two bounded workers perform password hashing/verification, with a 32-job queue. Admission completion rechecks room revision, membership, phase, reservation and capacity.

**Rationale**: Security-sensitive primitives warrant established libraries; a general auth framework is unnecessary. A two-worker memory bound protects the 20 Hz coordinator. OS randomness, not simulation RNG, creates server epochs, room IDs and credentials.

**Admission reconciliation**: A five-second host deadline includes queueing; cancellation/expiry/disconnect before commit discards later work. Retain private completed outcomes for 60 seconds with eight reserved slots/connection; replay/query the original request ID, never blindly resubmit after a client timeout. The coordinator serializes cancellation against commit and reports an already committed outcome rather than pretending rollback. Result-cache exhaustion returns Busy before mutation.

**Alternatives considered**: Plaintext password storage, ad hoc hashes, logging salted hashes and deriving credentials from seeds are rejected. Blocking password verification on the simulation thread risks all rooms. Durable secret storage is unnecessary for transient rooms.

## R4 - Protocol and legacy behavior

**Decision**: Introduce tagged JSON protocol version 2 with explicit `Hello`, directory, create, join and resume operations. Keep newline-delimited TCP and WebSocket JSON object transport, existing line bounds and gameplay payloads. Version-1 `Join` receives its existing `Rejected{BadVersion}` shape before closure; v2 clients report upgrade-required against old hosts rather than auto-joining. Public discovery advertises supported versions.

**Rationale**: Adding required room selection and response variants is not safely backward compatible merely because fields are optional. The specification explicitly supersedes feature 004's v1 promise, not game rules. Avoid a legacy default room with hidden membership semantics.

**Alternatives considered**: Pretending v1 remains fully compatible creates wrong-room risks and deserialization failures on unknown variants. Supporting a parallel legacy single-room service doubles lifecycle behavior and is outside this increment.

## R5 - Known-server discovery and remote joining

**Decision**: Operator configuration specifies stable server ID, label, TCP game address and browser origin for each known server. Net-side bounded workers request local-only directories over the same v2 TCP protocol; no new HTTP client is required. Poll every five seconds with two-second per-source timeout, at most four concurrent requests, maximum 16 known servers and maximum 128 returned rooms per source. Never follow another host's known-server list or redirects. Web exposes cached summaries through `GET /rooms` and local-only `GET /rooms?scope=local`.

**Timing budget**: Two seconds covers a complete source attempt including handshake, all pages and one cursor restart. One queued/in-flight poll per source and fair oldest-due scheduling bound queue wait. Failure detection is poll interval plus queue wait plus attempt timeout: seven seconds for the two-server guide, at most 13 seconds with 16 peers/four workers. This does not change the separate five-second healthy-directory availability target or block cached/local reads.

**Rationale**: Both terminal and browser share aggregation through the host. A missing remote source does not block local play. Explicit operator trust prevents arbitrary player-chosen fetches, and stable identities prevent duplicate-name confusion.

**Alternatives considered**: Browser fan-out requires CORS and duplicates logic. Recursive federation creates cycles and unbounded traffic. UDP/mDNS and public matchmaking are not requested.

**Remote handoff**: CLI connects directly to the destination game address. Browser opens the destination's configured web origin with only room/server identity in the URL fragment; the destination collects name/password and obtains any resume proof locally. It never transfers secrets in URLs. A source membership must be explicitly left before committing a switch; unsuccessful browsing/selection preserves it. A failed transfer after explicit leave offers retry/selection, not a fictional atomic transaction across hosts.

## R6 - Network protection

**Decision**: Retain current TCP transport on trusted development/LAN networks. Public browser deployment requires HTTPS/WSS termination; public terminal play requires an authenticated encrypted tunnel until native TLS is separately designed. Reject HTTPS-page handoff to HTTP and require explicitly configured origins. Reconnect credentials are memory-only in CLI and origin-scoped `sessionStorage` in web; unavailable storage gives a warning that page-reload recovery is unavailable, not a crash.

**Rationale**: A room password does not secure an unencrypted network. Existing plaintext transport must not be presented as internet-safe merely because passwords are hashed on the host. Browser reload recovery needs tab-scoped storage, not long-lived local storage.

**Alternatives considered**: Passwords in CLI argv or URLs leak through process lists/history. Full native TLS PKI and cross-origin credential delegation expand scope substantially.

## R7 - Significant-event stdout logging

**Decision**: Typed significant events are serialized as JSON Lines to stdout by one writer thread. Use a bounded 4,096-record queue, a monotonic event sequence and UTC epoch-millisecond timestamps. Emit room/member/owner/settings transitions, match transitions and selected significant gameplay events (elimination, role/team result), admission failures and discovery source health transitions. Never emit raw messages, passwords, proofs, password verifiers or complete observations. Ordinary movement and per-tick snapshots are excluded.

**Rationale**: A common event schema gives operators correlation without high-volume tick logs. Logging must not stall rooms when a stdout consumer stops. Stdout backpressure makes lossless logging and nonblocking bounded memory incompatible: overflow increments a counter and later emits `logging.records_dropped` with its exact count; writer failure reports once to stderr and records unhealthy status. Normal-load acceptance requires zero drops. Shutdown flushes for at most two seconds, reporting undrained count to stderr before exit.

**Alternatives considered**: File-only logging contradicts the stdout request. Printing in the simulation loop blocks matches. Unbounded channels risk memory exhaustion. A new logging framework is unnecessary for one schema and sink.

## R8 - Input parity and validation

**Decision**: Extend existing room setup/lobby and navigation layers; expose authoritative owner identity rather than inferring it from slot order. Preserve configurable bindings, selected-controller navigation, native/mobile password entry, portrait bottom-corner controls, landscape layout, pointer cancellation and persistent results until Return to Lobby. Tests cover new flows through each supported input surface.

**Rationale**: Features 005-007 are established behavior, not optional polish. New browser screens must suppress gameplay while text fields/dialogs are active and return focus reliably. Terminal directory/password handling stays in existing online/setup paths, not a second client stack.

**Alternatives considered**: Mouse-only directory controls and a separate mobile input implementation regress supported users. Client-derived ownership can disagree with reservations and reconnect transfer.

## Research Outcome

All questions above have concrete decisions. Tests must falsify isolation, admission races, expiry and secret exposure before broad integration. No runtime implementation was performed in this phase.