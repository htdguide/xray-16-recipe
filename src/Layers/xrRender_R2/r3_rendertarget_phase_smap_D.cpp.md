# src/Layers/xrRender_R2/r3_rendertarget_phase_smap_D.cpp

> Binds one sun cascade's slice — or the rain map — as the depth target for a directional
> shadow pass.

**Needs** — [`r2_rendertarget.cpp`](r2_rendertarget.cpp.md) · [`r2_types.h`](r2_types.h.md) · [`xrRender/light.h`](../xrRender/light.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: slice selection, target binding and a clear.

## Purpose

The directional counterpart of the spot atlas preparation. The sun's cascades share one
array texture, one slice each; the rain shadow map is a separate texture of its own size.
Both go through this entry point, distinguished by the sub-phase the caller passes, which
is why the rain map has a sub-phase index at all.

## `begin_directional_shadow`

**Contract** — binds the target the given sub-phase writes and clears it to the far value.
Disables stencil. Unlike the spot path it does not set a viewport, because a cascade
occupies its whole slice.

```text
FUNCTION begin_directional_shadow(sub_phase)
  IF sub_phase is the rain map
      bind the rain map as depth; clear it; set the viewport to the rain map's size
  ELSE
      select the cascade slice numbered by sub_phase
      bind the atlas as depth; clear it
  stencil off
```

**Invariants** — the sub-phase index doubles as the slice index for cascades, so the
enumeration's cascade values must stay contiguous from zero. The rain value sits past the
cascades for exactly this reason.

**Notes** — the rain branch sets a viewport and the cascade branch does not, because the
rain map has its own resolution while a cascade slice is the atlas's. The clear is
unconditional and per cascade, which is three full clears of the atlas per frame; with
array targets they are three slices of one texture, which is the cheaper arrangement and
the reason the array path exists.

## `begin_directional_shadow_translucent`

**Contract** — the coloured-mask pass for directional shadows: enables colour writes,
restores the full viewport, and clears the colour target to white (fully transmitting).
Only reachable when translucent shadows are enabled, which no shipped configuration does.
