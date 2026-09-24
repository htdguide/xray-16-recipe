# src/Layers/xrRender/DetailManager_VS.cpp

> The hardware grass path's static geometry: every model pre-replicated as many times as fit in one draw's worth of shader constants, with each copy stamped with the constant index that will transform it.

**Needs** — [`DetailManager.h`](DetailManager.h.md) · [`BufferUtils.h`](BufferUtils.h.md) · [`HWCaps.h`](HWCaps.h.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: the vertex record here is an exact byte layout with hand-quantized 16-bit fields, and the batch size is computed from a device capability.

## Purpose

The live grass path. Instead of transforming plants on the processor every frame, it uploads each model's geometry **once**, replicated N times, and then draws N instances at a time by filling a block of shader constants with N transforms. This is instancing, hand-built out of a pre-replicated vertex buffer and a constant array, on hardware that had no instancing.

This file is the static half: the buffers and the batch size. The per-frame half — filling the constant array and issuing the draws — lives in the per-generation renderer directories, because the constants it writes differ between the forward and deferred paths.

## The batch size

```text
header_registers = 10       # constants the material needs for itself
registers_per_instance = 4  # a transform plus wind parameters, per plant
batch = clamp((device vertex constant registers - 10) / 4, 0, 64)
```

**Invariants**

- The batch size is **discovered from the device**, and everything else in the file is sized from it: the vertex buffer holds every model replicated `batch` times, and the index buffer likewise. A device with fewer constants gets smaller batches and more draw calls; the geometry adapts automatically.
- The ceiling of 64 is not a hardware limit — it is the point past which the vertex buffer's size stops being worth the reduction in draw calls. Sixty-four copies of every model in the level is already several megabytes.
- The floor of zero is nominal: a device reporting fewer than fourteen vertex constants cannot run this path at all, and would have failed the capability check that selected it.
- Each replica's vertices carry the constant index they will read — the replica's ordinal times four. This is the entire mechanism: the vertex program reads its transform from `constants[vertex.constant_index]`, so a single draw over the whole replicated buffer draws `batch` differently-placed plants. A rebuild on an API with real instancing replaces the replication and the stamped index with an instance id and a structured buffer, and deletes this file's buffer construction entirely — but must keep the batch concept, because the per-frame half still uploads transforms in blocks.

## The vertex record

```text
RECORD HardwareDetailVertex          # 20 bytes, packed
  x, y, z        : real (32-bit)
  u, v           : int (16-bit, signed)   # texture coordinates, quantized
  t              : int (16-bit, signed)   # height within the model, 0 at the base, 1 at the top
  constant_index : int (16-bit)           # which constant block transforms this replica
```

```text
quantization scale = 16384
FUNCTION quantize(v) -> clamp(floor(v * 16384), -32768, 32767)
```

**Invariants**

- The quantization scale is 16384, not 32767. That fixes the representable range at ±2.0 with 14 fractional bits, rather than ±1.0 with 15. Texture coordinates on a grass model do leave the unit range — an atlas tile reached with a wrap — and the extra headroom is why the scale is a power of two below the maximum. The vertex program divides by the same constant.
- The `t` field is the vertex's height as a fraction of the model's own bounding-box height: 0 at the root, 1 at the tip. **This is the wind weight.** The vertex program displaces each vertex by the wind amplitude times `t`, so a plant bends about its base instead of sliding. It is computed here, at load, because it is a property of the geometry and never changes.
- `t` is computed as `position.y / (box_max.y - box_min.y)`, which is only "0 at the root" if the model's box floor is at y = 0. The authored models satisfy that; a model authored with its origin at its centre would bend about the wrong point. Nothing checks it.
- The index buffer replicates each model's indices `batch` times, each copy offset by one model's vertex count, so the whole replicated run is one index range. The offsets accumulate in 16-bit arithmetic and therefore wrap silently if a model's vertex count times the batch size exceeds 65536 — with a batch of 64 that caps a detail model at 1024 vertices. Nothing checks this either; the shipped models are two orders of magnitude below it.

## `hw_load()` / `hw_unload()`

**Contract** — compute the batch size, build and upload the replicated vertex and index buffers, declare the vertex layout, and resolve the named shader constants the per-frame half will write. The buffers are push-once — the host copy is discarded at upload — because nothing ever reads this geometry back. Unload releases the layout and both buffers.

**Invariants** — The vertex and index buffers are built for *every* model in the level at once, replicated, in one allocation each. The memory cost is `sum(model vertices) * batch * 20` bytes, which the load path logs. For a level with forty grass models this is a few megabytes, paid once, to avoid a per-frame processor transform of every blade of grass in view. That trade is the entire reason this path exists.
