# src/xrEngine/xrHemisphere.h

> Declares the baked hemisphere tessellation accessors; the substance is in [`xrHemisphere.cpp`](xrHemisphere.cpp.md).

**Needs** — [`xrHemisphere.cpp`](xrHemisphere.cpp.md)
**Used by** — [`Environment.cpp`](Environment.cpp.md) · [`xrHemisphere.cpp`](xrHemisphere.cpp.md)
**Tier floor** — T2.

## Purpose

Declares the surface described in [`xrHemisphere.cpp`](xrHemisphere.cpp.md).

Exported units:

- **`xrHemisphereVertices`** — the direction table for a quality level, and its length.
- **`xrHemisphereIndices`** — the triangle list for a quality level, and its length in
  indices. Defined for qualities 1 and 2 only.
- **`xrHemisphereBuild`** — visits every direction of a quality level, handing each an equal
  share of a given total energy, through a caller-supplied function and context.
