# src/Layers/xrRender/blenders/Blender_Shadow_Texture.cpp

> The material a model is drawn with while its silhouette is being rendered into a shadow texture: pure black, no texture, no depth.

**Needs** — [`Blender_Shadow_Texture.h`](Blender_Shadow_Texture.h.md) · [`Blender.h`](../Blender.h.md) · [`Blender_Recorder.h`](../Blender_Recorder.h.md) · [`Blender_CLSID.h`](../Blender_CLSID.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: it emits a pass description with no parameter block.

## Purpose

Half of the forward renderer's model-shadow scheme. A model that casts a shadow is drawn twice: once into a small off-screen texture with this material, which records only *where the model is* as a black silhouette, and then once onto the world with [`Blender_Shadow_World.cpp`](Blender_Shadow_World.cpp.md), which projects that texture back down. Internal — the engine constructs it, no shipped material names it. Forward renderer only.

## `Compile`

```text
FUNCTION compile(context)
  pass:
    depth test and write off, blend replace, lighting and fog off
    stage 0: the pass constant, selected, for both colour and alpha;
             no texture, no matrix, no per-stage constant
    the pass's shared constant := 0, every channel
```

**Invariants** — depth is off in both directions. The shadow texture is a coverage mask, not a depth map: overlapping parts of the model must produce the same black as a single layer, which they do because every fragment writes the same constant. With depth on, self-overlap would still work, but the target carries no depth buffer to test against.

**Notes** — Declares itself explicitly **not lightmappable**. Emitting the constant through a texture stage rather than clearing the target is what lets the silhouette be drawn with the model's own geometry and its own vertex programs, including skinning — the shadow of an animated model follows its pose for free.
