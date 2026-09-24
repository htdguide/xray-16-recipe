# src/Layers/xrRender/FTreeVisual.h

> Declares the wind-animated model: a mesh whose vertices are quantized into a fixed tile and whose lighting arrives as a scale and bias pair, in a still and a simplifying variant.

**Needs** — [`FBasicVisual.h`](FBasicVisual.h.md) · [`xrCore/FMesh.hpp`](../../xrCore/FMesh.hpp.md)
**Used by** — [`FTreeVisual.cpp`](FTreeVisual.cpp.md) · [`ModelPool.cpp`](ModelPool.cpp.md) · [`R_Backend_tree.cpp`](R_Backend_tree.cpp.md) · [`r__dsgraph_build.cpp`](r__dsgraph_build.cpp.md)
**Tier floor** — T1: the tree definition is read as a byte image and the quantization constant is part of the shipped vertex layout.

## Purpose

Declares the surface implemented in [`FTreeVisual.cpp`](FTreeVisual.cpp.md). Despite the name, this type carries anything that sways: trees, bushes, hanging cloth, flags.

## Exported units

- **`TreeVisual`** — the base: a mesh, a placement transform, and a lighting scale and bias triple. Its draw sets up the wind constants and draws nothing itself.
- **`TreeVisual_Static`** — draws the whole mesh. Most trees.
- **`TreeVisual_Progressive`** — draws a slide-window range chosen by level of detail, exactly as [`FProgressive.cpp`](FProgressive.cpp.md) does, but with the window table shared through the level rather than owned.
- **`LightingTriple`** (private) — static colour, hemisphere term, sun term: the engine's universal baked-lighting triple.

## The quantization constants

```text
tile  = 16
quant = 32768 / 16 = 2048
```

**Invariants** — A tree's vertex positions are stored as signed 16-bit integers in units of 1/2048 of a world unit, giving a representable range of ±16 metres about the model's own origin and a resolution of about half a millimetre. Sixteen metres is the size of the largest authored tree; half a millimetre is well below what is visible. Both numbers are frozen by the shipped model files. The reciprocal of the quantum is pushed to the vertex program as a scale constant.
