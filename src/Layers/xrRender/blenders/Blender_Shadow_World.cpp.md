# src/Layers/xrRender/blenders/Blender_Shadow_World.cpp

> The material that projects an already-rendered shadow silhouette back onto the world, by multiplying the frame where the silhouette is dark.

**Needs** — [`Blender_Shadow_World.h`](Blender_Shadow_World.h.md) · [`Blender.h`](../Blender.h.md) · [`Blender_Recorder.h`](../Blender_Recorder.h.md) · [`Blender_CLSID.h`](../Blender_CLSID.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: it emits a pass description with no parameter block.

## Purpose

The second half of the forward renderer's model-shadow scheme, the counterpart to [`Blender_Shadow_Texture.cpp`](Blender_Shadow_Texture.cpp.md). The engine draws the receiving geometry again with this material and the silhouette texture bound, and the pass darkens the frame wherever the silhouette says the model blocks the light. Internal; forward renderer only.

## `Compile`

```text
FUNCTION compile(context)
  pass:
    depth test ON, depth write OFF
    blend MULTIPLY: the frame is multiplied by what this pass produces
    stage 0: the silhouette texture ADDED to the vertex colour,
             taken from the material's first texture, no matrix
```

**Invariants**

- Depth testing on and writing off is what confines the shadow to surfaces that are actually there and stops it from occluding anything itself.
- The add against vertex colour is the **softening and fade** control. The silhouette is black where the model blocks and white where it does not; adding the vertex colour lifts the black toward white, so a receiving vertex with a bright colour receives a faint shadow and one with a dark colour receives a strong one. That is how the shadow fades with distance and with the caster's own light contribution — the geometry's vertex colours carry it, and the material has no parameter for it at all.

**Notes** — Multiplying into the frame rather than blending by alpha means the shadow darkens whatever is already there proportionally, which is the physically sensible behaviour for occluded light and which also makes overlapping shadows from two casters compound rather than replace one another.
