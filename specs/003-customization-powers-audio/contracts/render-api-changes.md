# Contract: WebAssembly Render API Changes

Base contract: `specs/002-web-client/contracts/wasm-render-api.md`. The command layout and stride (16) do not change; every command is still a quad, circle, text or triangle.

## New exports

| Export | Signature | Meaning |
|--------|-----------|---------|
| `eksplode_re_set_looks` | `(ptr: u32, len: u32) -> i32` | loads a JSON array of up to four looks (the wire form of `Look`); `0` ok, `1` not UTF-8, `2` not a JSON array. Missing or invalid entries use slot defaults; never fails for bad fields. Kept until replaced. |
| `eksplode_re_portrait3d` | `(slot: u32, width: f32, height: f32, phase: f32) -> u32` | commands of the semi-3D character card art for `slot`'s current look, fitted into `width` x `height` pixels; `phase` is time in seconds for the idle motion; works without a loaded game; returns the number of commands (0 for an invalid size) |

`eksplode_re_portrait` (flat head crop) stays for the menu logo.

## Look to sprite (match view)

- Body colour = the look's colour; helmet = the colour mixed with white; face and eye colours fixed; a very dark body gets orange eyes and buckle (the existing black character).
- `Female`: ponytail pixels beside the helmet and lashes on the eye row; `Male`: none.
- Head accessories (`Cap`, `TopHat`, `Crown`, `Headphones`) and face accessories (`Glasses`, `Sunglasses`, `Moustache`, `Mask`) are pixel overlays on named rows of the 12x13 template; every overlay stays within the sprite width plus 1 pixel per side.
- Colours are unique among players (host rule), so the four characters stay distinct.

## Semi-3D character card (`eksplode_re_portrait3d`)

- 2:1 isometric projection of cubes. Parts: ground shadow, body cube, head cube (left, right and top faces shaded 0.8, 0.6, 1.0), the pixel face drawn on the head's front-right face as skewed quads, accessory boxes, arms and feet as small cubes.
- Motion: a vertical bob of at most 4% of the height with period 2 s, and a yaw swap between two views every 6 s using `phase`; deterministic for the same `phase`.
- All points inside the given rectangle for any valid size; invalid sizes give no commands.

## Power icons in the match

Each kind is a framed badge on its tile (bonuses a white frame, maluses a dark red frame with a pulsing outline): roller (skate), flame, extra bomb (small bomb), punch (fist), slow (snail), hyper speed (bolt), small bomb (small bomb with a minus). Drawn from small pixel templates like the bomb, with a letter fallback for tiny scales.

## Bomb in flight and effects

- A bomb with `flight` is drawn lifted by an arc: lift = `sin(pi * progress)` times 0.5 tile, with `progress = 1 - hop_left / punch_hop_ticks`, and a small shadow on the square below.
- A player with an active malus gets a coloured pulse under the feet (slow blue, hyper yellow, small bomb red); a player with the punch gets a small fist mark in the HUD, not on the sprite.

## Tests that bind this contract

Skin per look (colour, gender, each accessory changes pixels; defaults equal the old four characters); ASCII template checks (widths); `set_looks` codes and leniency; 3D portrait bounds, determinism, difference per slot/colour/accessory, `phase` motion bounded; icons distinct per kind; flight arc height range; node tests for the new exports; stride unchanged; size limits (wasm at most 300 KB).

## Amendment A1: pixel-art icons and directional, animated sprites

- **Power icons**: each `PowerUpKind` (eight with `Kick`) is a 10x10 pixel template in `sprites.rs`, drawn with the same `runs` helper as the bomb, inside the white (bonus) or dark red (malus) frame. Below a small tile size (under 20 px) the letter of the previous version is drawn instead. Icons: extra bomb = small bomb, roller = rollerskate, flame = flame, punch = glove, kick = boot, slow = snail, hyper speed = lightning bolt, small bomb = tiny bomb with a minus.
- **Player views**: `look::grid(look, view, frame)` with `view` in `Down` (front), `Up` (back), `Left` (side; `Right` is `Left` mirrored) and `frame` in `Idle`, `StepA`, `StepB`. The back view has no face (hair or helmet all around, hats and headphones as from behind), the side view shows one eye, one lens or temple, the hat brim to the facing side and the moustache or mask in profile. Colour, gender look and accessories are kept in every view.
- **Choosing view and frame**: the renderer keeps, per slot, the last facing (from the position change between snapshots, initially Down). A slot that moved between the two snapshots is "walking": the step frame alternates with the distance walked (`floor(2 * progress)` through the tile, A then B), with a one-pixel bob on the step frames; otherwise `Idle`. No wire change.
- **Portraits** (lobby flat portrait, editor preview) keep the front idle frame; the isometric lobby character is unchanged.
- Commands keep stride 16 and the draw order rules.
