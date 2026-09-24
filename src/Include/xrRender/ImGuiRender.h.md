# src/Include/xrRender/ImGuiRender.h

> The draw half of the debug overlay backend: hand the toolkit's draw lists to the graphics device.

**Needs** — [`RenderFactory.h`](RenderFactory.h.md) · [`xrEngine/Device_imgui.cpp`](../../xrEngine/Device_imgui.cpp.md) · [Seam: Debug overlay UI](../../../SYSTEM-REQUIREMENTS.md#seam-debug-overlay-ui) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`RenderFactory.h`](RenderFactory.h.md) · [`dxImGuiRender.cpp`](../../Layers/xrRender/dxImGuiRender.cpp.md) · [`dxImGuiRender.h`](../../Layers/xrRender/dxImGuiRender.h.md) · [`dxUIShader.cpp`](../../Layers/xrRender/dxUIShader.cpp.md)
**Tier floor** — T1: it receives the toolkit's own draw-list structures by pointer and reads them as laid out; the toolkit defines the layout.

## Purpose

The debug overlay toolkit is a [given seam](../../../SYSTEM-REQUIREMENTS.md#seam-debug-overlay-ui): it owns the widget logic and produces draw lists — vertex and index arrays with per-command texture and clip rectangle — and the host supplies input and draws the lists. The engine implements the input half itself. This interface is the draw half, and it is the only part that must be written once per graphics backend.

One instance per process, created by the device during startup and destroyed during shutdown.

## State

```text
RECORD ImGuiRenderState
  font_atlas   : Texture          # uploaded from the toolkit's atlas at device create
  vertex_buf   : dynamic buffer
  index_buf    : dynamic buffer
  pass         : Material         # textured, alpha-blended, scissored, no depth
```

## `IImGuiRender`

### `on_device_create(context)`

**Contract** — takes the toolkit's context, uploads its font atlas to the device, creates the dynamic buffers and the drawing state. Called once during device startup, immediately after the instance is created from the factory.

**Notes** — Passing the toolkit's context here rather than letting the implementor reach a global is what allows the debug overlay to exist at all in a build where the renderer is a separately loaded module: the toolkit's global state lives in the executable, and the module must be told about it explicitly. A rebuild without separately loaded modules can drop the argument.

### `frame`

**Contract** — begins a toolkit frame on the renderer's side. Called once per frame from the main loop, before any widget code runs.

### `render(draw_data)`

**Contract** — draws one frame's accumulated draw lists. Called after all widget code, late in the frame, over the finished image. The implementor walks the lists, uploads the vertex and index data, and issues one draw per command with that command's texture and scissor rectangle.

**Invariants** — the draw data is valid only during this call; nothing may be retained.

### `on_device_reset_begin` / `on_device_reset_end`

**Contract** — bracket a device reset: release everything that does not survive one, then rebuild it. Distinct from create and destroy because the toolkit's context and its atlas *do* survive a reset; only the device-side resources do not.

### `on_device_destroy`

**Contract** — releases everything. Called before the instance is returned to the factory, which is before the device goes away.

### `copy(other)`

**Contract** — the [`FactoryPtr.h`](FactoryPtr.h.md) duplication hook. Meaningless for a singleton, present because the ownership policy demands it.

## Notes

The whole interface, and the seam behind it, may be omitted from a rebuild: the debug overlay is developer tooling and the player-facing UI is the engine's own. What it costs to omit is every in-game inspector.
