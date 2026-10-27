# Data Model: Controller and Keyboard Controls

This feature translates local device state into existing movement, bomb and punch actions. It adds no game-state fields and no network-protocol messages.

## Control Action

Represents one action supported by the current player input.

| Field | Values | Validation |
|-------|--------|------------|
| action | `move_up`, `move_down`, `move_left`, `move_right`, `bomb`, `punch` | Required; only these six actions are configurable |

## Web Keyboard Preferences

The saved local preference for one web player.

| Field | Type | Validation |
|-------|------|------------|
| version | integer | Must match the supported preference version; unknown versions use defaults |
| bindings | action-to-key-code map | Every action has at least one valid key; a key cannot conflict across actions |

The versioned browser storage key is `eksplode.re.controls.v1`. The default map preserves existing aliases: `ArrowUp`/`KeyW`, `ArrowDown`/`KeyS`, `ArrowLeft`/`KeyA`, `ArrowRight`/`KeyD`, `Space`/`KeyE`, and `KeyZ`/`KeyQ`. Rebinding an action replaces its aliases with the new key. Invalid entries fall back to the default for that action. Storage read/write failure must not prevent keyboard play.

## Terminal Slot Bindings

The current-run keyboard map for a local CLI player slot or a network-joining CLI player.

| Field | Type | Validation |
|-------|------|------------|
| slot | integer 0 through 3, or the single joining player | One map per local slot; the joining player has one map; omitted values keep current defaults |
| up, down, left, right, bomb, punch | terminal key | Exactly one key per action; key cannot conflict with another action/slot or reserved controls |

Bindings are supplied through CLI configuration for the current process and are not persisted. Invalid input is a usage error; it does not silently replace another slot's mapping. A joining player may configure only their own input mapping.

## Connected Controller (Runtime Only)

The controller currently visible to this web page.

| Field | Type | Validation |
|-------|------|------------|
| index | non-negative integer | Must identify a currently connected controller; revalidate against the latest device list |
| mapping | string | Must be `standard` for the default mapping |
| connected | boolean | False/disappeared controllers contribute no input |
| buttons, axes | browser gamepad state | Read fresh for each input sample; absent/short arrays are treated as neutral |

Controller choice is not persisted between visits because the connected index can be reused. If no controller is selected, select the first standard-mapped controller. Disconnect or selection change clears its held state.

## Effective Player Input

The value produced for one existing game input tick.

| Field | Type | Rule |
|-------|------|------|
| movement | optional direction | Non-neutral selected-controller direction takes precedence; otherwise current keyboard direction |
| bomb | boolean | True when either input source has a new press for this tick |
| punch | boolean | True when either input source has a new press for this tick |

The existing client message and server-side validation remain unchanged.