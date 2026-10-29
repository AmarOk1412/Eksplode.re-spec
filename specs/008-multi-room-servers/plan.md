# Implementation Plan: Multi-Room Servers and Discovery

**Branch**: `main` (unchanged) | **Feature key**: `008-multi-room-servers` | **Date**: 2026-10-28 | **Spec**: [spec.md](spec.md)

**Input**: Feature specification from `specs/008-multi-room-servers/spec.md` and the user's stdout logging, cross-server and input-preservation requirements.

**Note**: Setup reports the feature key as `BRANCH`; the actual Git branch remains `main`. This command stops after design. Generate implementation tasks separately with `/speckit-tasks`.

## Summary

Refactor the single-room net host into an authoritative room registry. Each room owns its lobby, settings, match, input buffers, bots and lifecycle; connections are explicitly outside or bound to one room. Add protocol-v2 directory/create/join/resume and owner operations, secure 60-second reconnection reservations, operator-configured known-server discovery and direct host handoff. Preserve all existing modes and web/terminal input surfaces. Emit typed significant events as secret-free JSON Lines on stdout without blocking simulation.

## Technical Context

**Language/Version**: Existing workspace Rust stable/edition 2021 and declared workspace MSRV; browser JavaScript ES modules, HTML/CSS and existing WebAssembly renderer.

**Primary Dependencies**: Reuse `serde`, `serde_json`, `tungstenite`, ratatui/crossterm and standard threads/channels. Add `getrandom` 0.2, `argon2` 0.5 and `subtle` 2 to net for OS credentials, password verification and constant-time proof comparison; validate MSRV before pinning. No async runtime, database, HTTP client or logging framework.

**Storage**: In-memory room/membership/discovery state; no restart persistence. Browser proofs in origin-scoped `sessionStorage`, never localStorage/URLs. CLI proofs in memory. Password verifiers remain server-private.

**Testing**: Rust unit/integration and fake-clock tests; existing Node built-in JS suites; mixed TCP/WebSocket tests; expanded CDP browser smoke with desktop/mobile screenshots and input assertions; physical controller/mobile checks when available. Inject credential sources, clock, discovery transport and event sink in tests.

**Target Platform**: Linux-first server/terminal; existing desktop browsers, iOS Safari, Firefox for Android and Android Chrome. Public browser service behind HTTPS/WSS; TCP terminal service on trusted networks or encrypted tunnels.

**Project Type**: Existing Rust workspace, authoritative game service, terminal and server-served web clients.

**Performance Goals**: Independent 20 Hz rooms; three rooms of four participants complete 20 rounds per mode with no isolation failures; input feedback p95 <=200 ms; healthy local directory/join p95 <=2 seconds; healthy listings remain available within five seconds of a remote source failure.

**Constraints**: Four participants per room; no gameplay rules in JS; no new unsafe code; 64 KiB wire frames; bounded command, discovery, password and log queues; owner permissions validated by host; no plaintext secrets in public state/logs; protocol-v1 clients rejected safely. Existing controls, powers, audio, mobile orientation and persistent results remain intact.

**Scale/Scope**: Default 32 rooms/server, hard configurable bound 1-128; maximum 16 known servers, 128 local rooms/source; directory pages of 32; default 256 admitted/control connections and existing web connection cap raised coherently. Three fully populated rooms are the minimum tested load, not a claim that all 128 rooms meet latency targets. Different games means existing modes, not a general game plug-in system. Distributed single-match execution is excluded.

## Constitution Check

*GATE: checked before research and again after Phase 1 design.*

| Principle/Gate | Pre-design | Post-design |
| --- | --- | --- |
| I. Specification-first | PASS: recovered spec precedes runtime work; FR-001-019 bound scope | PASS: design traces to that spec; no runtime implementation |
| II. Test-first and coverage | PASS: require failing tests for each changed behavior | PASS: validation matrix covers every acceptance scenario; full gates remain required before merge |
| III. Core/frontend separation | PASS: no rules in browser/CLI | PASS: admission and ownership in net; core simulation unchanged |
| IV. Deterministic simulation | PASS: preserve `step` and injected RNG | PASS: per-room ticks/seeds isolated; service clocks/randomness never feed core implicitly |
| V. Four-player multiplayer/bots | PASS: retain four-slot rooms | PASS: capacity and mode checks include bots/reservations; takeover preserves state |
| VI. Dual-interface parity | PASS: terminal and browser required | PASS: contracts include both plus features 005-007 |
| Platform, minimal dependencies, safety | PASS: retain Linux/workspace structure | PASS: three security dependencies justified in research; no new unsafe or crates |
| Authority/input validation | PASS: host decides all mutations | PASS: serialized admission, bounded work, stale revision rejection and secret redaction |
| Documentation/merge gates | PASS: updates required with implementation | PASS: README, WEB, PROTOCOL, SECURITY and DEVELOPMENT updates included; fmt/clippy/tests mandatory |

No Constitution exceptions. Protocol-v1 compatibility is an earlier feature promise, explicitly superseded in this spec; it is not a constitutional gameplay exception. These PASS results validate the design, not implementation test execution.

## Project Structure

### Documentation (this feature)

```text
specs/008-multi-room-servers/
  spec.md
  checklists/requirements.md
  plan.md
  research.md
  data-model.md
  contracts/room-protocol.md
  contracts/operations-and-navigation.md
  quickstart.md
```

`tasks.md` is intentionally deferred to `/speckit-tasks`.

### Source Code (repository root)

```text
game/crates/net/src/
  host.rs          coordinator, routing, scheduling and room lifecycle
  room.rs          existing room parsing plus net-owned room/membership policy
  protocol.rs      v2 wire messages and typed errors
  client.rs        unjoined connection, directory/admission and resume helpers
  codec.rs         retain bounded tagged JSON framing
  discovery.rs     proposed bounded known-server polling/cache
  events.rs        proposed typed event schema and bounded stdout writer
game/crates/net/tests/{room_host,loopback,lobby,codec,dev}.rs
game/crates/web/src/{lib,options,args,http,ws,assets}.rs
game/crates/web/assets/{app,room,nav,controls,touch}.js
game/crates/web/assets/{index.html,style.css}
game/crates/web/js-tests/{room,app,lobby,nav-focus,nav-input,nav-layers,osk,touch}.test.mjs
game/crates/web/tests/{room,mixed,lifecycle,http,websocket,origin,args,docs}.rs
game/crates/cli/src/{args,online,setup,lobby}.rs
game/crates/cli/tests/{args,online,setup,lobby,join_errors,pty}.rs
game/scripts/browser-smoke.mjs
game/README.md
game/docs/{WEB,PROTOCOL,SECURITY,DEVELOPMENT,ARCHITECTURE}.md
```

**Structure Decision**: Extend existing room/server/client modules and test homes rather than add crates or a parallel frontend. Only discovery and event-sink ownership justify dedicated net modules. `core::RoomSettings`, simulation and renderer remain gameplay-only. Existing core owner helpers remain usable for local-only play but must not authorize network actions; explicit net owner data drives online clients.

## Phase 0 - Research

Completed in [research.md](research.md): R1 coordinator/room state; R2 reconnect/ownership; R3 credential libraries and bounded work; R4 protocol negotiation; R5 known-server directory and handoff; R6 transport security; R7 stdout event semantics; R8 input parity. No unresolved clarifications remain.

## Phase 1 - Design and Contracts

- [data-model.md](data-model.md) defines server, room, membership, reservation, discovery and event state with validation and lifecycle transitions.
- [contracts/room-protocol.md](contracts/room-protocol.md) specifies v2 handshake, admission, routing, discovery, limits and failure semantics.
- [contracts/operations-and-navigation.md](contracts/operations-and-navigation.md) specifies options, stdout, security boundaries and all client input surfaces.
- [quickstart.md](quickstart.md) separates runnable baseline checks from post-implementation multi-server scenarios.

## Implementation Sequence and Validation

1. Write failing isolation/owner/reservation tests in existing net test files with injected clock. Extract per-room state and bounded coordinator scheduling while preserving single-room gameplay results; adapt in-process Host helpers to take explicit room IDs.
2. Add v2 codecs/handshake and unjoined `Client` connection path; update all first-party TCP, WebSocket and in-process callers together. Test v1 safe rejection, phase/room authorization, line limits and one controlling connection.
  User-approved sequencing adjustment: T009 establishes shared transport during foundations; first-party client cutover and legacy Join retirement happen with US1 T022-T024 after room-selection UI exists. Intermediate legacy availability is not a full-feature compatibility promise.
3. Implement create/admission/settings/removal, private password verification and room revision checks. Add five-second admission deadlines, cancellation/status reconciliation and bounded 60-second completed-result retention; test late completion, duplicate retries, concurrent final-seat requests, stale verification results, mode/capacity conflicts and server-limit rejection.
4. Implement disconnect/bot takeover, acknowledged proof rotation with a ten-second same-operation replay window capped by the original reservation deadline, fake-clock expiry, owner reservation/transfer and empty-room shutdown. Test lost resume replies/acknowledgments, unchanged retry deadlines, elimination/rematch return and A-disconnects/B-leaves reservation preservation.
5. Add known-server workers, local-only query and cached aggregation; expose web directory without peer-origin relaxation. Bound each full attempt to two seconds with fair one-poll-per-source scheduling; test failure detection against interval + queue + timeout (seven seconds in the guide, at most 13 seconds with 16 peers), separately from five-second healthy-directory availability. Retain same-name, incompatibility, oversize, nonrecursive-fetch and destination-race tests.
6. Emit typed significant events at mutation decisions through the injected sink and bounded stdout writer. Require zero loss at normal three-room load; under saturation/output failure count undelivered records and report exact loss on recovery without blocking gameplay. Test category coverage, field allowlists, secret canaries, sequence/overflow, recoverable/broken pipe and shutdown behavior.
7. Extend browser/CLI setup, directory, password, owner controls, reconnect and handoff; authoritative owner identity replaces slot inference. Apply feature 005-007 navigation/touch contracts and update startup defaults: `--bots` fills each explicitly created room, not an invisible startup room, clamped to its capacity.
8. Extend existing browser smoke and user docs; map every spec acceptance scenario to an automated test. Run three-mode load and 100 lifecycle cycles, then full quality gates. Record physical-device limitations instead of claiming unperformed checks.

## Complexity Tracking

No Constitution violations. The protocol-major transition and three security dependencies are deliberate, documented costs; speculative federation, persistent accounts, live migration and a replacement networking runtime are excluded.
