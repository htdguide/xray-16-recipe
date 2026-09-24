# src/Include/xrRender/ThunderboltDescRender.h

> The renderer's half of one authored lightning bolt: load and hold its model.

**Needs** — [`RenderFactory.h`](RenderFactory.h.md) · [`ThunderboltRender.h`](ThunderboltRender.h.md) · [`xrEngine/thunderbolt.h`](../../xrEngine/thunderbolt.h.md)
**Used by** — [`RenderFactory.h`](RenderFactory.h.md) · [`ThunderboltRender.h`](ThunderboltRender.h.md) · [`dxThunderboltDescRender.cpp`](../../Layers/xrRender/dxThunderboltDescRender.cpp.md) · [`dxThunderboltDescRender.h`](../../Layers/xrRender/dxThunderboltDescRender.h.md) · [`thunderbolt.h`](../../xrEngine/thunderbolt.h.md)
**Tier floor** — T2: model ownership, two calls.

## Purpose

A weather configuration names several lightning bolts, each with its own model, its own sound, and its own sky-flash colour. The engine parses those definitions; the renderer loads each one's model. One instance per definition, created and destroyed through the [render factory](RenderFactory.h.md) with the definition.

## State

```text
RECORD ThunderboltDescRenderState
  model : optional<Model>   # none until create_model, none again after destroy_model
```

## `IThunderboltDescRender`

```text
FUNCTION create_model(model_name : text)
FUNCTION destroy_model()
FUNCTION copy(other)
```

**Contract** — `create_model` is called once while the bolt's configuration section is parsed, with the model path read straight from that section. `destroy_model` releases it. The model is a static mesh, not a skeleton; the animation of a strike is in its material, not its geometry.

**Notes** — The pairing with [`ThunderboltRender.h`](ThunderboltRender.h.md) is the pattern this whole directory repeats: **a per-definition interface that owns resources, and a per-system interface that draws.** It appears again for weather keyframes and for lens flare elements. A rebuild that keeps the pattern keeps the property that makes it worth having — device resources are acquired once per authored definition, at parse time, and released once, so a device reset walks a known list rather than chasing live effects.
