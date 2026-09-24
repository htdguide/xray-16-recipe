# src/Include/xrRender/LensFlareRender.h

> The renderer's half of the sun's lens flare: the materials of each flare element, and the draw of the sun disc, the flare chain and the screen gradient.

**Needs** — [`RenderFactory.h`](RenderFactory.h.md) · `xrEngine/LensFlare.h` · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`RenderFactory.h`](RenderFactory.h.md) · [`dxLensFlareRender.cpp`](../../Layers/xrRender/dxLensFlareRender.cpp.md) · [`dxLensFlareRender.h`](../../Layers/xrRender/dxLensFlareRender.h.md) · [`dxThunderboltRender.cpp`](../../Layers/xrRender/dxThunderboltRender.cpp.md) · [`thunderbolt.h`](../../xrEngine/thunderbolt.h.md) · [`xr_efflensflare.h`](../../xrEngine/xr_efflensflare.h.md)
**Tier floor** — T2: material ownership and three draws.

## Purpose

A lens flare is authored as a sun disc, a chain of flare sprites strung along the line from the sun's screen position through the screen centre, and a full-screen gradient that brightens when the sun is near the view direction. The engine owns the authored parameters, the sun's world direction, and the occlusion test that decides how much of the flare is visible. The renderer owns each element's material and does the drawing.

Two interfaces because there are two granularities: one per flare *element* (the disc, each sprite, the gradient each have their own material), and one per flare *system*.

## State

```text
RECORD FlareElementRenderState
  material : optional<Material>    # none until create_material

RECORD LensFlareRenderState
  quad : Geometry                  # the screen-space quad every element is drawn on
```

## `IFlareRender` — one element

```text
FUNCTION create_material(material_name, texture_name)
FUNCTION destroy_material()
FUNCTION copy(other)
```

**Contract** — resolves one element's material and releases it. Created and destroyed through the [render factory](RenderFactory.h.md), held by the element it belongs to. That is all an element needs from the renderer: the geometry is a screen-space quad the system builds.

## `ILensFlareRender` — the system

```text
FUNCTION on_device_create()
FUNCTION on_device_destroy()
FUNCTION render(flare, draw_sun : bool, draw_flares : bool, draw_gradient : bool)
FUNCTION copy(other)
```

**Contract** — `render` draws the three parts of the effect, each independently switchable. The three flags are how the engine expresses partial visibility: with the sun behind geometry the disc is suppressed but the gradient may still show, and a user setting disables the flare chain alone.

The implementor reads the sun's screen position, its occlusion fraction and each element's placement from the flare object it is handed.

**Notes** — Splitting the draw into three switchable parts rather than three calls keeps the state changes together — all three share one blending mode and one depth setting — and lets the implementor skip the whole thing cheaply when all three are off, which is the common case indoors.

The occlusion fraction the engine supplies comes from an occlusion query against the sun's disc, which means **the flare's brightness lags the geometry by however many frames the query takes to resolve**. That latency is visible when stepping quickly behind a pillar and is inherent to the technique, not a bug to fix.
