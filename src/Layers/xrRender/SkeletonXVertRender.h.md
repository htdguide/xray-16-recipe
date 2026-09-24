# src/Layers/xrRender/SkeletonXVertRender.h

> The vertex layout software skinning writes into the dynamic stream — and the one decision it encodes: tangent frames do not survive the processor path.

**Needs** — [`SkeletonX.cpp`](SkeletonX.cpp.md) · [`R_DStreams.h`](R_DStreams.h.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`SkeletonX.cpp`](SkeletonX.cpp.md) · [`SkeletonX.h`](SkeletonX.h.md) · [`SkeletonXSkinXW.h`](SkeletonXSkinXW.h.md) · [`SkeletonXSkinXW_CPP.cpp`](SkeletonXSkinXW_CPP.cpp.md) · [`SkeletonXSkinXW_SSE.cpp`](SkeletonXSkinXW_SSE.cpp.md)
**Tier floor** — T1: it is a byte layout handed to the graphics device as a stride.

## Purpose

Declares the output of software skinning. It is a separate file for one reason: both skinning implementations and the mesh that calls them need the layout, and none of them should need each other.

## State

```text
RECORD vertRender                # packed to 2-byte alignment; 32 bytes
  position : 3 reals
  normal   : 3 reals
  u, v     : real
```

**Invariants**

- **Tangent and bitangent are absent, although every source vertex format carries them.** The deferred renderer always skins on the device, so a tangent frame is never needed on this path; only the oldest forward renderer falls back to processor skinning, and it does not use tangent frames at all. A rebuild that software-skins for a renderer which *does* need tangents must extend this record and both skinning implementations together.
- Thirty-two bytes, which is two aligned four-component groups — the layout is chosen so that a 4-wide implementation can emit each vertex as exactly two stores. See [`SkeletonXSkinXW_SSE.cpp`](SkeletonXSkinXW_SSE.cpp.md).
- The matching packed format identifier is declared alongside the mesh: position, normal, one texture coordinate pair.
