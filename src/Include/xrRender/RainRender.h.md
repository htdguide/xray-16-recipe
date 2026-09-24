# src/Include/xrRender/RainRender.h

> The renderer's half of the rain effect: draw the drops, and tell the engine how big one drop's bounding volume is.

**Needs** — [`RenderFactory.h`](RenderFactory.h.md) · [`xrEngine/Rain.h`](../../xrEngine/Rain.h.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`RenderFactory.h`](RenderFactory.h.md) · [`dxRainRender.cpp`](../../Layers/xrRender/dxRainRender.cpp.md) · [`dxRainRender.h`](../../Layers/xrRender/dxRainRender.h.md)
**Tier floor** — T2: one draw and one geometry query.

## Purpose

The engine simulates rain: a population of drops around the camera, each one spawned above, falling, and retired when it hits geometry — the hit test is a ray against the collision database, and a hit also spawns a splash and may place a wallmark. The renderer owns the drop's model and material and draws the population.

## State

```text
RECORD RainRenderState
  drop_model : Model       # the stretched-quad drop mesh and its material
  bounds     : Sphere      # invariant: the drop model's own bounding sphere
```

## `IRainRender`

```text
FUNCTION render(rain)                    # draw the whole live drop population
FUNCTION drop_bounds() -> Sphere         # the drop model's bounding sphere
FUNCTION copy(other)
```

**Contract** — `render` reads the live drops' positions and orientations from the effect object it is handed and draws them as instanced stretched quads.

`drop_bounds` returns the bounding sphere of the drop's *model*, which the engine needs on its own side: it uses the radius to decide how far above the camera to spawn drops, and to size the ray it casts for the hit test. This is the only geometric fact about the drop that crosses the interface.

**Notes** — The bounds query is the interesting decision. The drop's visual size is content — it comes from the drop model the renderer loaded — but the *simulation* has to agree with it or drops will visibly pass through surfaces before they splash. Rather than duplicating the number in configuration, the engine asks the renderer. A rebuild should keep that direction of dependency: the thing that owns the art owns the size, and the simulation asks.

The rain effect is drawn after the world's opaque pass and before the clouds, with depth testing on and depth writes off.
