# src/Include/xrRender/ThunderboltRender.h

> The renderer's half of the lightning effect: draw the current bolt.

**Needs** — [`RenderFactory.h`](RenderFactory.h.md) · [`ThunderboltDescRender.h`](ThunderboltDescRender.h.md) · [`xrEngine/thunderbolt.h`](../../xrEngine/thunderbolt.h.md)
**Used by** — [`RenderFactory.h`](RenderFactory.h.md) · [`ThunderboltDescRender.h`](ThunderboltDescRender.h.md) · [`dxThunderboltRender.cpp`](../../Layers/xrRender/dxThunderboltRender.cpp.md) · [`dxThunderboltRender.h`](../../Layers/xrRender/dxThunderboltRender.h.md) · [`thunderbolt.h`](../../xrEngine/thunderbolt.h.md)
**Tier floor** — T2: one draw.

## Purpose

The engine schedules lightning: it picks a bolt definition, a direction, a moment and a duration from the weather state, and drives a brightness envelope over its life. The renderer draws the chosen bolt's model, oriented towards its strike direction, plus the sky flash.

## State

`Stateless.` The bolt's model belongs to its definition; the live bolt's transform and brightness belong to the engine.

## `IThunderboltRender`

```text
FUNCTION render(thunderbolt)      # draw the live bolt, if any
FUNCTION copy(other)
```

**Contract** — reads the current bolt, its transform and its brightness from the effect object it is handed. Draws nothing when no bolt is live. Called once per frame from the weather pass, after the sky and before the clouds.

**Notes** — There is no device create or destroy pair here, unlike its siblings, because the bolt models are owned per definition by [`ThunderboltDescRender.h`](ThunderboltDescRender.h.md) and this interface holds nothing of its own. The asymmetry is correct rather than an oversight.
