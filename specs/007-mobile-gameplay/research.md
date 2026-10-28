# Research: Mobile Browser Gameplay

## Responsive Match Orientation

- **Decision**: Detect orientation with the browser media query for layout only. Portrait is a supported match orientation with the joystick at bottom left and bomb/power actions at bottom right; landscape keeps the existing horizontal control layout. Do not lock orientation or block input based on orientation.
- **Rationale**: Players can use their phones naturally in portrait, while landscape users retain the established wide layout.
- **Alternatives considered**: Force landscape using browser orientation controls (contradicts the requested vertical mobile play).

## Mobile Browser Detection

- **Decision**: Detect touch capability from primary coarse pointer, any coarse pointer, `navigator.maxTouchPoints`, or touch-event support; do not allowlist browser names. Include Firefox for Android in mobile smoke coverage.
- **Rationale**: A fine primary pointer does not mean the device lacks touch, and browser-name checks are brittle. Capability detection covers Firefox Android and hybrid touch devices.
- **Alternatives considered**: User-agent checks (version-specific and easy to omit); primary `(pointer: coarse)` alone (can miss a touchscreen when a mouse is primary).

## Touch Gesture Model

- **Decision**: Use Pointer Events and track pointer identifiers. The left zone owns one active joystick pointer and captures it for the gesture; right-side action buttons handle independent pointers. `pointerup`, `pointercancel`, lost capture, blur and page hiding all release state.
- **Rationale**: Pointer Events provide a common browser path for touch and other pointing devices, expose stable pointer identity for multi-touch, and support pointer capture when a finger moves outside its starting area. This fits the project's no-dependency HTML/JavaScript client.
- **Alternatives considered**: Separate Touch and Mouse event handlers (duplicate behavior and compatibility-event risks); relying on pointer hover or click alone (cannot represent held movement).

## Movement Mapping

- **Decision**: Treat the left touch zone as a virtual joystick. Movement follows the thumb's displacement from its touch origin, with a central neutral area; the strongest axis selects one of the existing Up, Down, Left or Right directions. Release or cancellation returns to neutral.
- **Rationale**: The current client/network input carries one optional cardinal direction per tick. Mapping the gesture to that existing value enables touch movement without changing game rules, messages or server validation.
- **Alternatives considered**: Add diagonal movement (requires a game-input and rules change, outside this feature); four independent directional buttons (does not meet the requested joystick-like touch zone).

## Action Mapping and Input Integration

- **Decision**: Provide distinct right-side Bomb and Punch controls. Each action is a one-shot press, not a repeated action while a finger remains down. Sample touch state in the existing 50 ms input interval. Preserve existing source precedence for movement by using selected controller first, active touch direction second, and keyboard direction as fallback; combine bomb and punch as one-shot flags from all sources.
- **Rationale**: This preserves the established input cadence and client message shape while allowing a thumb on each side to move and act concurrently. No new timer or network state is needed.
- **Alternatives considered**: A separate touch polling loop (can race the existing input tick); continuous bomb/punch while held (changes action semantics); extending the wire protocol (not required by the existing `Input` message).

## Conditional Punch Control

- **Decision**: Show Bomb during mobile gameplay and show Punch only while the local player's latest observation has the Punch power.
- **Rationale**: The action area reflects the player's current abilities and avoids offering an unavailable action.
- **Alternatives considered**: Always show Punch disabled (adds clutter); always show it active-looking (misrepresents availability).

## Match Layout and Browser Gestures

- **Decision**: In portrait, place the joystick at bottom left and bomb/power controls at bottom right; in landscape preserve the current horizontal layout. Show controls for touch-capable devices, apply `touch-action: none` only to controls, and account for safe-area insets in both orientations.
- **Rationale**: Corner placement keeps both thumbs reachable and leaves the arena center visible; limiting gesture suppression to controls avoids disabling page behavior elsewhere.
- **Alternatives considered**: Disable browser gestures across the whole page (unnecessarily affects setup and desktop use); place controls over the arena center (obscures gameplay).

## Testing and Compatibility

- **Decision**: Test joystick mapping, touch-capability detection including `ontouchstart`, conditional Punch visibility, one-shot actions, pointer ownership and cancellation as Node tests; verify embedded module/result markup through Rust HTTP tests; exercise Firefox-like detection, portrait/landscape layouts, touch actions and result persistence in browser smoke; retain manual checks on iOS Safari, Firefox Android and Android Chrome.
- **Rationale**: Pure input behavior is deterministic and CI-friendly, while real devices are still needed to verify viewport changes, safe areas and browser touch delivery.
- **Alternatives considered**: Hardware-only testing (not repeatable in CI); synthetic pointer tests as the only evidence (do not prove actual mobile viewport and browser behavior).
