# Data Model: Animated Lobby Scene

**Date**: 2026-11-06
**Related**: [spec.md](spec.md), [research.md](research.md), [contracts/lobby-scene.md](contracts/lobby-scene.md)

All entities are local presentation state. No persisted data, schema migration, player slot, room message, or gameplay entity is added. Names below are proposed design names, not existing APIs.

## LobbyTimeline

| Field | Type or Domain | Meaning |
|-------|----------------|---------|
| rng | existing core Rng | Initialized from an injected unsigned 32-bit seed widened to u64 |
| deck | seven VignetteKind values | Each kind appears exactly once, in shuffled order |
| cursor | integer 0 through 6 | Active item in the deck |
| local_tick | integer 0 through 299 | Active time inside the current five-second item at 60 Hz |
| previous_kind | optional VignetteKind | Last completed kind; prevents identical deck-boundary neighbors |

`reset(seed)` constructs the same deck and initial scene for the same seed. `advance(ticks)` accepts 0 through 15 ticks; zero is a no-op, larger values are rejected without changing state. On local tick 300, clear all authored objects and select the next item at tick zero; refill and shuffle only after the deck is exhausted. No unbounded global tick counter is needed. Sampling and painting never consume RNG or advance the timeline.

One deck takes 2,100 ticks (35 seconds). A required cue tied to a fixed offset in its kind recurs within 3,900 ticks (65 seconds) across any adjacent decks. Keep cue offsets and item durations fixed; colours may differ by authored actor but do not change coverage. Tests verify the bound across adversarial deck positions and sample actual frame tags, not only deck labels.

## VignetteDefinition

| Field | Type or Domain | Meaning |
|-------|----------------|---------|
| kind | one of seven values below | Stable testable identity |
| duration_ticks | exactly 300 | Includes entrances, action, resolution, and clearing |
| actors | up to four authored actor tracks | Fixed looks, depth, travel, and poses |
| bombs | up to three authored bomb tracks | Drop/push/incoming timing and visual progression |
| effects | bounded timed effects | Local smoke, blast, or squash effects; no screen flashes |
| coverage_tags | set of authored motifs | Observable events with known tick offsets for coverage tests |

| Kind | Guaranteed Visible Action and Outcome |
|------|---------------------------------------|
| RunDrop | Rightward run, visible bomb drop and fuse, safe exit, local explosion |
| SameLevel | Lead and another distinguishable runner on the foreground baseline, same-direction travel, safe exit |
| CounterRunBlast | Foreground opposing runners, lead character caught in a bomb explosion, non-graphic elimination |
| BackgroundRun | Foreground lead plus smaller runners on a separate background baseline, including same and opposing directions |
| KartLeft | Side-view rider and kart enter from the right and exit left |
| KartRight | Side-view rider and kart enter from the left and exit right |
| PushSquash | Character visibly pushes one bomb; a second incoming bomb briefly flattens that character; both bombs and the effect clear |

Authored event offsets must place the first moving character on screen within 60 ticks and complete every action by tick 299. The next scene begins immediately; no separate blank intermission is needed. Entrance/exit positions can extend beyond the logical stage, but the complete sampled scene is clipped to its bounds. The cartoon contact is a scripted cue, not collision detection or a new interpretation of bomb rules.

## SceneFrame

| Field | Type or Domain | Meaning |
|-------|----------------|---------|
| kind | VignetteKind | Source of this sample, useful to native tests |
| phase | local_tick plus interpolation | Finite normalized time, no wall clock |
| actors | bounded list of DecorativeActor | Background actors precede foreground ones |
| bombs | bounded list of DecorativeBomb | Includes pushed/incoming bombs where authored |
| effects | bounded list of DecorativeEffect | Localized, short-lived, non-flashing |
| coverage_tags | currently visible motifs | Enables testing visible action rather than scheduling alone |

The logical stage is 640 by 200 units. Drawing uniformly fits that space to the requested backing-buffer dimensions; resizing changes only projection, not time, RNG, or choreography. Canvas backing dimensions are finite, positive, and at most 8,192 per axis. All emitted command coordinates/colours are finite. Background art and actors are emitted before foreground art because the current painter respects command order rather than layer metadata.

## DecorativeActor

| Field | Type or Domain | Meaning |
|-------|----------------|---------|
| identity | local actor index | Exists only within a vignette, not a network player ID |
| look | existing Look value | Authored fixed appearance independent of saved or participant looks |
| direction | Left or Right | Determines facing and travel |
| depth | Background or Foreground | Baseline and scale; background is smaller and behind the foreground |
| position | finite logical x/y | Derived from a timed track |
| pose | Idle, StepA, StepB, Push, Ride, Squash, Gone | Side-profile presentation |
| scale and opacity | bounded finite values | Enable depth, brief flattening, and fade without flashes |

Typical authored transitions: running steps alternate until exit; push alternates steps while the first bomb moves; incoming-bomb contact changes Push to Squash to Gone; a blast changes running to a brief squash/fade to Gone; Ride stays seated while kart wheels animate. These transitions do not emit core events.

## DecorativeBomb and DecorativeEffect

A bomb has a local identity, role (Dropped, Pushed, Incoming), finite position, optional authored fuse interval, and visual state (Absent, Fuse, Moving, Blast, Cleared). Moving bombs use timed positions, not velocity integration or physics. Every dropped explosive bomb has a readable fuse and blast; pushed/incoming gag props may clear after contact without a separate explosion.

Effects have kind (LocalBlast, Smoke, Flatten), start/end ticks, bounds, and opacity. Their lifetimes end before vignette rollover. Effects never cover the whole page, flash more than three times per second, play audio, or call match event handlers.

## SceneController

Browser-only lifecycle state; choreography remains in Rust.

| Field | Type or Domain | Meaning |
|-------|----------------|---------|
| seed | u32 | Sampled once per page visit, injected into the renderer; fixed values usable in tests |
| user_paused | boolean | Internal deterministic-fixture state; no player-facing pause control |
| reduced_motion | boolean | Live media-query preference, separate from user pause |
| menu_visible | boolean | Derived from screens(state).mainMenu and rendered visibility |
| covered | boolean | An active navigation/modal layer obscures the main menu |
| document_visible | boolean | Inverse of document.hidden |
| renderer_ready | boolean | All optional scene exports are available |
| failed | boolean | Scene unavailable or a scene call failed; use PNG |
| raf_handle | optional browser handle | At most one outstanding scene callback |
| previous_timestamp | optional finite time | Browser scheduling only, reset whenever inactive |
| fractional_ticks | value in [0, 1) | Converts active browser time into 60 Hz tick deltas |
| last_alpha | value in [0, 1) | Holds a stable painted sample when explicitly paused |

Activity requires menu_visible, document_visible, not covered, renderer_ready, not failed, not reduced_motion, and not user_paused. Repeated synchronization with identical inputs is idempotent. Cancel the queued callback and reset previous_timestamp when becoming inactive. Hidden time is discarded, not stored as future debt. Cap any individual active RAF delta at 250 ms, advancing at most 15 ticks; this avoids bursts after a suspended browser callback. If time goes backwards, use zero elapsed time.

| Event | Transition and Visible Result |
|-------|-------------------------------|
| Initial menu | PNG immediately; initialize timeline once when graphics are ready; then begin drawing |
| Test-fixture pause | Preserve current frame, tick, and fractional time; cancel scene RAF; set user_paused |
| Test-fixture resume | Clear user_paused; draw from preserved time if otherwise active |
| Reduced motion enabled | Cancel RAF and show representative PNG; preserve timeline and user_paused |
| Reduced motion disabled | Resume preserved time only if user_paused is false and visibility permits |
| Screen hidden, modal opened, or tab hidden | Freeze timeline and cancel RAF; no catch-up on return |
| Return to visible menu | Repaint preserved sample, establish new RAF origin, keep one loop |
| Resize | Repaint preserved sample at the new size; never reset or advance timeline |
| Export missing, invalid response, or scene exception | Show PNG, stop scene RAF, leave application/network state untouched |
| Page teardown | Cancel callback and detach controller-owned media/visibility/resize handlers |

The user revision removes the pause/play button and its DOM wiring. No scene-specific focus target is added. PNG and canvas are aria-hidden and nonfocusable; the existing game heading provides identity. Internal pause state remains available for test fixtures and is not stored as a player preference.

## Isolation and Validation Invariants

- Resetting, advancing, or drawing the scene cannot modify observation/interpolation state, actual looks, room state, picking geometry, saved settings, audio, or protocol traffic.
- Scene state may share the existing command output buffer, but not the interpretation of a gameplay frame.
- Invalid sizes, interpolation, or excessive tick advances return a bounded error/empty result without panic or partial timeline mutation.
- All state is page-local; reloading discards pause and the scene seed. No schema or storage-key change is needed.