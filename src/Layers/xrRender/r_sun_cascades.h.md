# src/Layers/xrRender/r_sun_cascades.h

> The per-cascade record of the sun's shadow: the projection it was rendered with, the rays that defined its extent, its size and its depth bias.

**Needs** — [`light.h`](light.h.md)
**Used by** — [`light.h`](light.h.md) · [`r2_R_sun_support.h`](../xrRender_R2/r2_R_sun_support.h.md)
**Tier floor** — T2: two small records; nothing device-facing.

## Purpose

The sun is a directional light and cannot use one shadow map — the near ground and the far horizon need wildly different texel densities. The standard answer is *cascades*: several shadow maps covering nested slices of the view, each fitted to its slice. This header declares what one cascade remembers.

## State

```text
RECORD Ray
  origin    : vector3
  direction : vector3

RECORD Cascade
  transform   : matrix4      # the light-space projection this cascade was rendered with
  rays        : list<Ray>    # the bounding rays whose hull the projection was fitted to
  size        : real         # the cascade's extent in world units
  bias        : real         # the depth bias applied when sampling it
  reset_chain : bool         # this cascade's fit is stale; refit it and every one after
```

Invariants:

- `rays` is the working set the fit is computed from, kept per cascade rather than recomputed, because the fit is incremental: a cascade whose rays have not changed keeps its transform, and keeping it is what stops the shadow map's texel grid from sliding under static geometry and making shadow edges crawl.
- `reset_chain` propagates *forward*: cascades are nested, so a cascade whose fit is invalidated invalidates every coarser one. A rebuild must honour the direction — invalidating backwards would refit the near cascade for a far change and reintroduce the crawl.
- `bias` is per cascade, not global. Each cascade has a different world-units-per-texel ratio, and a single bias that hides acne in the near cascade produces visible peter-panning in the far one.

**Notes** — The header is only the shape. Which slices the cascades cover, how the rays are chosen, how the fit is computed and how the cascades are blended at their boundaries all live in the deferred path's sun code. What belongs here, and what a rebuilder needs from this file alone, is that a cascade is a *remembered* fit with an explicit staleness flag, not a value recomputed every frame.
