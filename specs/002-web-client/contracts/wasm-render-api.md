# Contract: WebAssembly Render API

Module: `eksplode_re_render.wasm`, built from `crates/web-render` for `wasm32-unknown-unknown`. It has **no imports** (no JS glue, no wasm-bindgen) and exports linear `memory` plus the functions below. All integers are little-endian `u32`/`i32`, all floats `f32`.

## Exports

| Export | Signature | Meaning |
|--------|-----------|---------|
| `memory` | linear memory | JS copies input into it and reads commands out of it |
| `eksplode_re_alloc` | `(len: u32) -> u32` | reserves `len` bytes, returns the pointer (0 if `len` is 0 or too large, max 1 MiB) |
| `eksplode_re_free` | `(ptr: u32, len: u32)` | releases a block from `eksplode_re_alloc` |
| `eksplode_re_load` | `(ptr: u32, len: u32) -> i32` | parses UTF-8 JSON of a `core::Observation`; `0` ok, `1` not UTF-8, `2` invalid JSON/shape, `3` arena too large (> 64x64) |
| `eksplode_re_frame` | `(width: f32, height: f32, alpha: f32) -> u32` | builds the draw commands for a canvas of `width` x `height` pixels and `alpha` in 0.0-1.0 between the previous and the current tick; returns the number of commands (0 if nothing is loaded) |
| `eksplode_re_output_ptr` | `() -> u32` | pointer to the command buffer (valid until the next `eksplode_re_frame`) |
| `eksplode_re_stride` | `() -> u32` | floats per command (16) |
| `eksplode_re_pick_tile` | `(x: f32, y: f32) -> i32` | optional: tile index `y * width + x` under a canvas point, `-1` if none (used for hover hints) |

## Command layout (stride 16)

`[kind, r, g, b, a, x0, y0, x1, y1, x2, y2, x3, y3, size, text, layer]`

| kind | Meaning | Fields used |
|------|---------|-------------|
| 0 | filled quadrilateral | colour, points 0-3 |
| 1 | filled circle | colour, `x0,y0`, `size` = radius |
| 2 | one text character, centred | colour, `x0,y0`, `size` = font px, `text` = ASCII code |
| 3 | filled triangle | colour, points 0-2 |

Commands are ordered back to front; JS paints them in order with `fillStyle = rgba(...)`. Coordinates are canvas pixels; the view is scaled and centred to fit `width` x `height` with a 16 px margin.

## Draw list content per frame (top-down pixel art)
1. The board: a darker front edge strip, then a light rectangle for the whole arena (`layer` -1).
2. Then, sorted by `(layer kind, row, column)`: walls (grey tile with a white window), blocks (grey tile with dark-red brick lines), flames (orange, yellow and pale core squares), power-ups (white-framed coloured badge with the letter `B`, `R` or `S`, bobbing), bombs (pixel bomb, pulsing size, blinking spark), players (pixel-art character, a soft shadow under the feet).
3. Characters are drawn in four skins by slot: 0 blue with a white helmet, 1 black with orange eyes, 2 red, 3 purple.
4. Every quad is snapped to whole canvas pixels, and the scale is a whole number, so pixels and tiles stay crisp and neighbours share edges.
5. Eliminated players are not drawn.

`eksplode_re_portrait(slot, width, height, rows) -> u32` fills the same command buffer with one character fitted into `width` x `height` pixels (quads only, `layer` 0): the whole figure for `rows` 0, otherwise the first `rows` rows zoomed so the head fills the card. It works without a loaded game and is used for the lobby cards and the menu logo.

## Error handling
- No export panics on bad input: invalid JSON returns a code, an oversized or zero allocation returns 0, `frame` with nothing loaded returns 0.
- The module is built with `panic = "abort"`; any unexpected trap makes the page show "graphics module failed" and stop rendering (it never affects the server).

## Versioning
`eksplode_re_stride()` and the exported function set are the contract; changing the layout bumps the stride or adds an export. The JS asserts `eksplode_re_stride() === 16` at start-up and shows a clear error when the page and the module are out of sync.

## Tests that bind this contract
Native Rust tests call the same functions (the FFI shims are plain functions on native targets); `node --test` instantiates the committed `.wasm`, loads a fixture observation JSON, and checks counts (4 cubes with distinct colours, expected walls), ordering (back to front), determinism (`frame` twice gives equal buffers) and every error code.

## Feature 003 additions

Feature 003 adds exports (`eksplode_re_set_looks`, `eksplode_re_portrait3d`), power icons and flight
drawing. The source of these changes is `specs/003-customization-powers-audio/contracts/`
(`render-api-changes.md`); this contract keeps its meaning for everything else.
