# Mobile Touch Controls Contract

## Orientation and Layout

- Room setup and lobby remain usable in portrait and landscape.
- A touch-capable player can play in either orientation; no orientation lock or rotate prompt is used.
- Portrait places the joystick in the bottom-left corner and bomb/power actions in the bottom-right corner.
- Landscape retains the existing horizontal match layout.
- An orientation or viewport change updates the layout and clears held inputs; gameplay remains active.

## Control Zones

| Orientation | Movement | Actions |
|-------------|----------|---------|
| Portrait | Virtual joystick at bottom left | Bomb at bottom right; Punch appears there only while the local player has the Punch power. |
| Landscape | Existing horizontal movement zone/layout | Existing horizontal action layout; Punch appears only while powered. |

The joystick selects one of Up, Down, Left or Right; its center is neutral. Bomb is a one-shot action. Punch is a one-shot action and is absent while unavailable. No game rules or network messages are added.

## Pointer Lifecycle and Multi-Touch

- Each touch is associated with its originating zone and pointer identifier.
- The joystick tracks at most one pointer; other touches cannot steal or redirect it.
- A touch in an action zone can activate its control while joystick movement continues on multi-touch devices.
- Pointer release, cancellation, lost capture, blur, document hiding, navigation-layer opening or orientation change clears affected held input.
- Touching controls does not scroll, zoom or select page content during gameplay; gesture suppression is limited to controls.

## Compatibility

Touch capability is detected from coarse-pointer media queries, available touch points or touch-event support, not a browser-name allowlist. Firefox for Android is supported. Existing desktop keyboard, mouse and controller behavior remains unchanged.

## Round Results

- The result view names the winner, draw or mode-specific outcome and lists every participant's final standing and available final stats.
- The scoreboard remains visible after the server broadcasts the lobby state.
- Return to Lobby is an accessible, touch/keyboard/controller-operable action that dismisses results and shows the latest lobby.
- No cumulative points or kill-count scoring is introduced; rows use the final outcome and stats already in the final observation.
