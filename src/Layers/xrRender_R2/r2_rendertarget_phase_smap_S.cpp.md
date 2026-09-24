# src/Layers/xrRender_R2/r2_rendertarget_phase_smap_S.cpp

> Prepares the spot shadow atlas: clears it once per batch, then re-aims the viewport at
> one light's rectangle before that light's casters are drawn.

**Needs** — [`r2_rendertarget.cpp`](r2_rendertarget.cpp.md) · [`SMAP_Allocator.h`](SMAP_Allocator.h.md) · [`xrRender/light.h`](../xrRender/light.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: target and viewport binding, and a clear.

## Purpose

The atlas is one texture shared by a batch of lights. Two entry points: one clears it and
binds it, called once when a batch starts; one re-points the viewport at a single light's
rectangle, called before each light's casters. A third handles the translucent-shadow
variant, in which a coloured mask is written instead of depth.

## `clear_shadow_atlas`

**Contract** — selects atlas slice zero, binds it as the depth target (with the colour
surface only where the device demands one), and clears it to the far value. Called once
per batch. On one backend it also fully resets the device's bound state, because a
preceding parallel context may have left bindings the driver would otherwise carry over.

## `begin_spot_shadow`

**Contract** — binds the atlas and sets the viewport to this light's rectangle, so that
everything drawn lands inside it and nothing escapes into a neighbour's. Draws front faces
only. Stencil off. Colour writes off wherever the device can render depth alone.

```text
FUNCTION begin_spot_shadow(light)
  select atlas slice 0
  bind the atlas as the depth target
  viewport = (light.rect.x, light.rect.y, light.rect.side, light.rect.side), depth 0..1
  cull back faces
  stencil off
  IF the device can compare-sample depth THEN colour writes off
```

**Invariants** — the viewport is the *only* thing restricting the draw to the light's
rectangle; there is no scissor. A caster whose projection leaves the rectangle is clipped
by the viewport, which is correct — it is outside the light's frustum by construction.

**Notes** — front faces rather than back is a choice against peter-panning: rendering the
near surfaces means the depth stored is the first occluder, and the bias applied at lookup
time pushes toward the light. The source notes an unexploited optimization: when several
lights cover more than half the same casters, their shadow passes could share geometry
submission. Nothing does this.

The slice index is fixed at zero because spot lights use a single-slice atlas; the slice
machinery exists for the sun's cascade array. The source notes that rendering several
lights into *different* slices in parallel would raise the batch size, and does not do it.

## `begin_spot_shadow_translucent`

**Contract** — the second pass over a light's casters, in which translucent casters write
a colour mask instead of depth, so that stained glass tints the shadow. Enables colour
writes; for a cube-face sub-light it simply clears the rectangle to white, for a real spot
it draws a full-rectangle quad with the light's own mask material.

**Invariants** — this path is incomplete in the shipped source: it asserts on entry that
the buffer clear for the translucent pass is unimplemented. Translucent shadows are
therefore *off* in every shipped configuration, and the option that enables them is
command-line only. A rebuild should either implement the per-light rectangle clear — which
is the missing piece, since clearing the whole atlas would destroy the neighbours' masks —
or drop the feature.
