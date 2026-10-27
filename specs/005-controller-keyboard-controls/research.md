# Research: Controller and Keyboard Controls

## Browser Controller Access

- **Decision**: Use the browser's built-in Gamepad API from the web client. Accept the standard gamepad mapping; display a useful unavailable message and retain keyboard input if the API is blocked, absent, or the controller is not standard-mapped.
- **Rationale**: The controller is a browser input device, not a game rule. Native browser support avoids a runtime dependency and preserves the existing client-to-server input contract. The official Gamepad guidance notes that a pre-connected device may not become visible until the user interacts with it while the page is visible; sample fresh state by device index rather than holding an old state object. The Gamepad Permissions Policy defaults to `*`, but a restrictive embedding policy can block `getGamepads()` and the connection events.
- **Alternatives considered**: A controller library adds a dependency for a small standard mapping; Rust/WASM integration would move presentation input away from the existing JavaScript client and complicate testing; custom mappings for arbitrary controllers exceed the Xbox-compatible scope.
- **References**: [MDN: Using the Gamepad API](https://developer.mozilla.org/en-US/docs/Web/API/Gamepad_API/Using_the_Gamepad_API), [MDN: Permissions-Policy gamepad directive](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Permissions-Policy/gamepad).

## Mapping and Sampling

- **Decision**: Map the standard left stick and D-pad to movement, Xbox A (standard button 0) to bomb, and Xbox B (standard button 1) to punch. Use a stick dead zone of 0.25 and sample controller state on the existing 50 ms input cadence. Treat bomb and punch as rising-edge actions so a held button does not repeat on every tick.
- **Rationale**: The existing player input accepts one direction and one-shot bomb/punch actions. This mapping works with the standard Xbox layout while remaining independent of game rules. A threshold filters small analog drift; sampling on the existing tick avoids a second animation loop.
- **Alternatives considered**: D-pad only excludes analog movement; continuous held-button handling would repeatedly trigger one-shot actions; a high-frequency requestAnimationFrame loop is unnecessary for the existing 20 Hz action cadence.

## Input Source Arbitration

- **Decision**: When a selected controller has a non-neutral direction, use that direction; otherwise use the current keyboard direction. Combine keyboard and controller bomb/punch presses as one-shot booleans for the next input tick. Clear both trackers on blur, hidden page, controller disconnect and controller selection change.
- **Rationale**: Players may mix devices. Controller movement must not be masked by a stale held keyboard key, and either device should be able to trigger an action. Clearing state matches existing keyboard blur behavior and prevents stuck actions.
- **Alternatives considered**: Keyboard-first precedence makes a held key unexpectedly override a controller; last-event ordering is difficult to represent consistently for analog input and complicates repeatable tests.

## Web Settings and Persistence

- **Decision**: Add a controls panel and an importable `controls.js` presentation module. Store keyboard bindings in a versioned key (`eksplode.re.controls.v1`) using the existing sanitize/load/save pattern. Preserve current aliases as defaults: arrows and WASD, Space and E, Z and Q. A rebind replaces the aliases for that action; collisions and reserved browser/game keys are rejected without changing the prior map. Controller selection is session-local and defaults to the first connected standard-mapped controller.
- **Rationale**: Existing web settings modules use small pure helpers, versioned storage, validation and storage-failure fallbacks, with Node tests. The panel provides the single requested place for keyboard settings and controller selection. Connected indices can be reused after disconnect and device IDs are not suitable as a durable preference.
- **Alternatives considered**: Put all logic in `app.js` (less modular and harder to test); persist controller IDs across sessions (unreliable and unnecessary); store settings server-side (unneeded and would couple personal controls to multiplayer state).

## Terminal Configuration

- **Decision**: Add optional per-slot launch bindings for local CLI play and one local binding set for a network-joining CLI player, retaining all current mappings when unspecified. Parse and validate the bindings in the existing CLI argument layer and share the configured mapper with local sessions and online joins. Preferences apply only to that CLI run; web keyboard preferences remain the persistent settings.
- **Rationale**: The terminal already has a centralized four-slot mapper used by both local and online play, plus a manual CLI option parser, but no terminal preference storage convention. Per-run options satisfy terminal customization without adding an unrelated configuration-file subsystem or changing the server input message.
- **Alternatives considered**: Add a new persistent CLI configuration file (new file format, path policy and lifecycle beyond the requested web panel); modify core or network input messages (unnecessary because controls already map to existing actions).

## Test and Hardware Strategy

- **Decision**: Unit-test keyboard mapping, conflicts, storage sanitation, standard controller mapping, dead zone, one-shot edges, disconnect, selection and source arbitration with fake Gamepad-shaped values. Test CLI default/custom mapping and malformed/duplicate options. Then manually validate the already-connected Xbox-compatible device in a visible, focused Linux browser on the local server.
- **Rationale**: Automated browser smoke tests can inject keyboard events but cannot prove physical hardware exposure. The controller may appear only after a button press or stick movement while the page is visible; testing must separate OS visibility from browser API visibility.
- **Alternatives considered**: Headless-only controller verification is insufficient; requiring an external controller test application does not validate the actual Bomberust page.