# src/Include/xrRender/DebugRender.h

> The batched line channel and the small slice of raw renderer state that debug overlays are allowed to touch.

**Needs** — [`DebugShader.h`](DebugShader.h.md) · [`DrawUtils.h`](DrawUtils.h.md) · [`xrAPI/xrAPI.h`](../xrAPI/xrAPI.h.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`xrAPI.h`](../xrAPI/xrAPI.h.md) · [`dxDebugRender.cpp`](../../Layers/xrRender/dxDebugRender.cpp.md) · [`dxDebugRender.h`](../../Layers/xrRender/dxDebugRender.h.md) · [`debug_renderer.h`](../../xrGame/debug_renderer.h.md)
**Tier floor** — T2: a drawing interface; the whole file compiles away in a shipping build.

## Purpose

[`DrawUtils.h`](DrawUtils.h.md) draws shapes one at a time and ships in the editors. This interface is the other half, and it exists only in a debug build: it *accumulates* line geometry across a frame and draws it once, and it exposes a handful of renderer state controls that debug overlays need and nothing else should have.

The physics layer is its main customer — a rigid-body world visualized as wireframes is thousands of segments per frame, and drawing them individually is unusable.

## State

```text
RECORD DebugRenderState
  pending_lines : accumulated vertex/index pairs with colours   # cleared at frame end
  shaders       : list<DebugShader>[SHADER_COUNT]               # created on demand
```

**Invariants** — accumulated lines live exactly one frame: whatever was added between one frame end and the next is what is drawn.

## `IDebugRender` — what an implementor must provide

### `add_lines`

**Contract** — appends a batch of line segments: an array of positions, an array of index pairs into it, and one colour for the whole batch. Copies what it is given — the caller's arrays may go away immediately. Does not draw.

**Notes** — Taking positions and index pairs rather than segment endpoints is the load-bearing choice: a wireframe box is eight positions and twelve pairs, not twenty-four positions. The physics layer's wireframes are dense enough that the difference is real.

### `render`

**Contract** — draws everything accumulated since the last frame end, as one batch, with the currently set material and state. Called once per frame from the debug overlay pass.

### `on_frame_end`

**Contract** — discards the accumulation. Separate from `render` so that a frame in which nothing chose to draw the overlay still clears the queue instead of letting it grow.

### Renderer state passthrough

```text
FUNCTION set_material(shader : DebugShader)
FUNCTION set_world_transform(m : Matrix)
FUNCTION set_cull_mode(mode)            # off / clockwise / counter-clockwise
FUNCTION set_depth_test(enabled : bool)
FUNCTION set_ambient(colour : int (32-bit))
FUNCTION next_scene_mode()
```

**Contract** — each forwards directly to the renderer's state cache. They are the minimum a debug overlay needs: choose a material, place geometry, decide whether it is occluded by the world, and tint it.

**Notes** — `set_depth_test(false)` is what makes an overlay visible through walls, which is the whole point of most of them. `next_scene_mode` cycles the renderer's scene visualization — normals, overdraw, wireframe and so on — and is bound to a console command; it is here rather than on the renderer interface because nothing outside debug tooling may call it.

The ambient colour override exists so that unlit debug geometry is visible in a dark scene without the overlay having to know how lighting works.

### Debug material management

```text
ENUM DebugShaderHandle { window }      # exactly one, today
FUNCTION set_debug_material(handle)
FUNCTION destroy_debug_material(handle)
```

**Contract** — a tiny named pool of materials the debug layer uses, created on first use and released on demand. The enumeration has a single member — a translucent panel material for drawing backdrops behind debug text — and a count sentinel.

**Notes** — A named pool with one entry is a fixture that never grew. A rebuild should let the debug layer own a material like anything else and delete this; the only reason it is here is that a debug material must be created by the renderer module and released before the device goes away, and this was the cheap way to give the debug layer exactly one.

### `draw_triangle`

**Contract** — one triangle, under a transform, in a colour, drawn immediately rather than accumulated. The one shape the line channel cannot express.

## Notes

The whole interface is compiled out of a shipping build, along with every call site. A rebuild may keep it always present and gate it at run time — the cost is a virtual call per debug draw in a build that makes none — which is the more maintainable choice and costs nothing measurable.

Both this interface and [`DrawUtils.h`](DrawUtils.h.md) reach the caller through the global environment, and both are owned by the renderer module. A backend additionally registers its debug renderer with the physics layer's visualization hook at startup and unregisters it at shutdown; that registration, not the interface, is what makes rigid bodies visible.
