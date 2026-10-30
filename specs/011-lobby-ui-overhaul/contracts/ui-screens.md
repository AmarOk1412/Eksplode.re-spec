# Contract: Web Client Screen Inventory

Replaces/extends the relevant panels in `crates/web/assets/index.html` (ids are current; final markup is
an implementation detail of `/speckit-tasks` + implementation, this contract fixes the *screen graph and
required elements*, not exact HTML).

## Screen graph

```text
Main Menu
├── Create new game  → Create-room form (existing #room-panel content, unchanged fields)
│                        → on submit → Lobby
├── Join a game      → Room list (existing #directory-panel content: list + Refresh + New room)
│                        → New room → Create-room form
│                        → select room → (existing join/password flow, unchanged) → Lobby
└── Options          → Options (tabbed: Customize | Sound | Controls)
                         (also reachable from the in-game pause menu's Sound/Controls entries,
                          landing on the matching tab)
```

## Lobby screen required elements

- Header row: room name/mode summary (existing), **configuration icon** (owner-only, opens Configuration
  panel), **leave icon** (any player).
- Four seat cards (existing `#slots`), each either:
  - Occupied (human or bot): shows name, ready state (existing), and a **cross control** — visible only
    to the owner, and only meaningful for that seat (kicks the human via `RemovePlayer`, or removes the
    bot via `RemoveBot{slot}`).
  - Empty: shows a **"+" control**, visible to any player, that calls `AddBot{slot}`.
- Action row (bottom), exactly two buttons in this order: **Ready** (everyone), **Start** (owner only;
  disabled/hidden for non-owners, consistent with `Lobby::can_start`'s existing reason text for why it's
  disabled).
- Explicitly removed from this screen: "Add bot," "Remove bot," "Change look" buttons; room
  name/mode/size/time/capacity/friendly-fire/password controls (moved into the Configuration panel).

## Configuration panel (opened from the lobby's configuration icon; owner-only)

- Contains exactly the fields already in `RoomSettings` (name, mode, size, minutes, friendly_fire) plus
  password set/clear — i.e., the content currently inline in `#lobby-panel` per the existing survey of
  `index.html` lines 126-161 — moved wholesale into this panel, gated by the existing `roomControls()`
  ownership check (`crates/web/assets/room.js`).
- Saving sends one `SetRoom` request (contracts/protocol-changes.md); canceling discards local edits and
  closes without sending anything.
- Non-owners never see this icon/panel (FR-012).

## Options screen (tabbed)

- Tab bar: Customize, Sound, Controls (in that order, matching the order listed in the spec).
- Sound tab: existing sound panel content (music/effects volume, mute, test sound), unchanged.
- Controls tab: existing controls panel content (key bindings, controller select), unchanged.
- Customize tab: see below.
- Opening Options from the main menu or from the in-game pause menu's Sound/Controls entries both land on
  this same tabbed screen (pause menu's Sound entry opens with the Sound tab active, Controls entry with
  the Controls tab active, consistent with today's direct-panel behavior).

## Customize tab (replaces today's `#editor`)

- Character preview image/canvas (existing colour-swatch + live preview), with a **small randomize icon**
  anchored to its bottom-right corner (replacing the current full-size "Random" button).
- Selectors: Head, Face, Costume (new). **No gender/"Look" selector.**
- No separate "Random" button outside the preview image (FR-006).
- "Done"/close behavior unchanged from today's editor.

## Scoreboard (`#results-panel`)

- Table header: exactly **Player, Kills, Bombs placed** (replacing today's Player/Result/Bombs/Range/
  Speed/Powers).
- Row order: per data-model.md's sort key (last-alive/winner first, then reverse elimination order, with
  the documented deterministic tie-break) — replacing today's lobby-slot-order (`resultRows()` in
  `crates/web/assets/app.js`).

## Focus styling (`crates/web/assets/style.css`)

- The `:focus-visible` rule at (today) line 125-127 changes its `outline` color away from `#d43b32` (red)
  to a non-red, sufficiently visible color consistent with the existing palette (e.g., reuse an existing
  accent/ink color already in the stylesheet's custom properties) — outline presence/offset otherwise
  unchanged, so keyboard/controller focus visibility (accessibility) is preserved.
