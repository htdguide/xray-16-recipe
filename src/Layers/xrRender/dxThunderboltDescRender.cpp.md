# src/Layers/xrRender/dxThunderboltDescRender.cpp

> Loads and holds the one mesh a lightning-bolt description draws with.

**Needs** — [`Include/xrRender/ThunderboltDescRender.h`](../../Include/xrRender/ThunderboltDescRender.h.md) · [`dxThunderboltDescRender.h`](dxThunderboltDescRender.h.md) · [`Include/xrRender/RenderDetailModel.h`](../../Include/xrRender/RenderDetailModel.h.md) · [`xrCore/FS.h`](../../xrCore/FS.h.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`dxThunderboltDescRender.h`](dxThunderboltDescRender.h.md)
**Tier floor** — T1: it hands a whole file image to the detail-model loader, which reads a frozen on-disk layout.

## Purpose

A weather configuration declares several *thunderbolt descriptions* — a named lightning variant with a model, two gradient sprites and a sound. The engine half owns the naming, the timing and the random choice among them. This file owns the single device-facing part: the bolt's geometry.

The split exists because the engine half must load on a dedicated server, which has weather and no device; the mesh may only be loaded where a renderer exists.

## State

```text
RECORD ThunderboltDescRenderer
  model : DetailModel   # owned; the bolt's mesh, with its own material
```

Invariant: the model is owned. It is created by `create_model` and released by `destroy_model`, and nothing else may free it. The renderer that draws the bolt ([`dxThunderboltRender.cpp`](dxThunderboltRender.cpp.md)) reads this field directly for the vertex and index counts and the material — it does not copy the mesh.

## `create_model`

**Contract** — Given a mesh name from the weather configuration, open that file under the shared-meshes root, hand the whole stream to the detail-model loader, and keep the result. Blocks on file I/O. A missing file is fatal: the configuration named a model that must exist, and there is no sensible fallback for "the lightning has no shape".

```text
FUNCTION create_model(name : text)
  stream = open("$game_meshes$", name)
  FAIL WITH "empty lightning_model" IF stream is none
  model = load_detail_model(stream)
  close(stream)
```

**Notes** — The detail-model format is the same one the grass layer uses: a small mesh with a material reference, a fixed vertex layout and a built-in scheme for being stamped repeatedly into a shared dynamic buffer with a per-copy transform, colour and texture-coordinate shift. Lightning reuses it precisely for that stamping ability — see the bolt renderer, which uses the shift to animate the bolt's texture.

## `destroy_model`

**Contract** — Release the mesh through the same loader that made it. Safe to call when nothing was loaded.

## `copy`

**Contract** — Adopt another descriptor companion's contents. Copies the model *reference*, not the mesh — the weather system duplicates descriptor objects when it rebuilds a weather cycle, and the duplicate points at the same loaded mesh. A rebuild must therefore not free the mesh from a copy; ownership stays with whichever object was given `create_model`.
