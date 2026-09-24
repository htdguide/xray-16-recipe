# src/Layers/xrRender/dxImGuiRender.cpp

> The renderer's filling of the debug-overlay port: it hands the overlay toolkit the graphics device and lets the toolkit's own backend do the drawing.

**Needs** — [`dxImGuiRender.h`](dxImGuiRender.h.md) · [`Include/xrRender/ImGuiRender.h`](../../Include/xrRender/ImGuiRender.h.md) · [`R_Backend.h`](R_Backend.h.md) · [Seam: Debug overlay UI](../../../SYSTEM-REQUIREMENTS.md#seam-debug-overlay-ui) · [Seam: Allocator](../../../SYSTEM-REQUIREMENTS.md#seam-allocator)
**Used by** — [`dxImGuiRender.h`](dxImGuiRender.h.md)
**Tier floor** — T2: it forwards six lifecycle calls. What stops T3 is nothing intrinsic.

## Purpose

The debug overlay is a third-party immediate-mode toolkit that ships its own renderer backends. The engine does not draw its widgets; it only has to (a) let the toolkit allocate through the engine's allocator, (b) give it the device handle, and (c) call its new-frame and draw entry points at the right moments and rebuild its device resources when the device is lost.

That is the whole file. Its value to a rebuild is the *order* of those calls and the allocator hookup, not any algorithm.

## `on_device_create(overlay_context)`

**Contract** — installs the engine's allocator as the toolkit's allocator, adopts the supplied overlay context as the current one, names the backend for the toolkit's own diagnostics, and initializes the toolkit's backend against the live graphics device. Must run after the device exists.

**Invariants**

- The allocator must be installed **before** the context is adopted and before the backend initializes, because both allocate. Installing it afterwards leaves early allocations owned by a different heap and the eventual free crosses heaps.
- The overlay context is *created by the engine* and handed in, not created here. The engine owns it because the overlay's input half lives on the engine side; this file is only the draw half.

## `frame()` / `render(draw_data)`

**Contract** — `frame` tells the toolkit's backend a new frame is starting; `render` submits a finished draw-data structure. Both are direct forwards. The engine calls `frame` once per frame before any widget code runs, and `render` once after.

## `on_device_reset_begin()` / `on_device_reset_end()`

**Contract** — release and recreate the toolkit's device-side resources (its font atlas texture, its buffers, its pipeline state) around a device loss.

**Invariants** — these bracket every device reset. Skipping the release leaks the old device's resources; skipping the recreate leaves the overlay invisible until the next reset.

## `set_state(draw_data)`

**Contract** — sets the viewport to the overlay's display size. Nothing else.

**Notes** — The rest of this routine is present only as a commented block describing the full state the toolkit's backend sets for itself: input layout, vertex and index buffers, the four shader stages, a blend state, a depth-stencil state and a rasterizer state. It is there as documentation of what the engine would have to do if it ever drew the overlay itself. A rebuild that supplies its own overlay backend needs exactly that list.

## `copy(other)`

**Contract** — assigns one instance over another. Exists because the port declares it for the renderer-swap path; this filling has no state, so the copy is trivial.
