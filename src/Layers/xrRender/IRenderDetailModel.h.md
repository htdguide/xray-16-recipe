# src/Layers/xrRender/IRenderDetailModel.h

> The renderer-side shape of one grass or debris model: its source geometry, its bounds, its material, and the one operation the detail layer needs — stamp a transformed copy into a shared vertex stream.

**Needs** — [`Include/xrRender/RenderDetailModel.h`](../../Include/xrRender/RenderDetailModel.h.md) · [`DetailModel.h`](DetailModel.h.md) · [`Shader.h`](Shader.h.md)
**Used by** — [`DetailModel.h`](DetailModel.h.md)
**Tier floor** — T1: the two vertex records here are exact byte layouts handed to the graphics device, and the transfer operation writes into a mapped device buffer.

## Purpose

The detail layer draws tens of thousands of small objects — grass tufts, stones, litter — every frame. They are never drawn individually. A few dozen distinct *models* are instantiated thousands of times each by copying their vertices into one large shared buffer with the instance's transform and colour already baked in. This interface is what one such model must be able to do to participate.

It is a separate interface from the concrete model because the copy operation has two implementations chosen at startup — one that bakes the transform on the CPU and one that leaves the vertices untouched and lets a vertex program do the work — and the detail manager holds the model through this interface without knowing which it got.

## State

```text
RECORD DetailModelSource
  bounding_sphere : sphere        # used for the per-instance visibility and fade decision
  bounding_box    : box
  flags           : bit set       # authored per model; chiefly: does this model wave in the wind
  min_scale       : real          # the authored scale range; an instance picks uniformly within it
  max_scale       : real
  material        : shader        # every instance of this model draws in one pass list
  vertices        : list<SourceVertex>
  indices         : list<int (16-bit)>
```

```text
RECORD SourceVertex        # what the model file holds
  position : vector3
  u, v     : real

RECORD StreamVertex        # what goes into the shared draw buffer
  position : vector3
  colour   : int (32-bit, packed)   # per-instance lighting and the wind phase, see below
  u, v     : real
```

**Invariants**

- Indices are 16-bit and are *offsets within this model*, not within the shared buffer. The transfer operation is given a base offset to add. This is what caps a detail model's vertex count and, more importantly, caps how many instances of one model fit in a single draw: the shared buffer's index space must not exceed the 16-bit range between flushes. The detail manager's batching is sized from that.
- The scale range is authored per model and is the whole of the size variation in the grass layer. There is no per-instance authored scale anywhere; instances are placed on a grid and vary only by this range and their rotation.
- The material is per *model*, not per instance. Instances of one model are batched precisely because they share it.

## `transfer(transform, destination_vertices, colour, destination_indices, index_offset)`

**Contract** — writes one transformed instance of this model into a shared vertex and index stream. Appends exactly this model's vertex and index counts; the caller has already reserved that much room. Writes into a mapped device buffer, so it must not read back what it wrote and must not allocate or block. Called tens of thousands of times per frame, single-threaded, in the innermost loop of the detail layer.

```text
FUNCTION transfer(transform, out_vertices, colour, out_indices, index_offset)
  FOR EACH v IN vertices
    out_vertices.append(StreamVertex{
      position = transform applied to v.position,
      colour   = colour,
      u = v.u, v = v.v })
  FOR EACH i IN indices
    out_indices.append(i + index_offset)
```

## `transfer(transform, destination_vertices, colour, destination_indices, index_offset, du, dv)`

**Contract** — the same, with a texture-coordinate offset added to every vertex. The offset selects one tile out of an atlas, which is how several visually distinct grass types share one texture and therefore one draw.

**Notes** — The packed colour is not a colour in the usual sense. It carries the instance's baked-in lighting — sampled from the level's light map at the instance's position, so grass under a tree is dark — and, on the wind-animated path, the instance's phase offset so that neighbouring tufts do not sway in lockstep. What exactly each channel means is the contract between this stamping code and the detail material's vertex program, and it differs between the two implementations. It is the one place in the detail layer where a rebuild must read both sides together before changing either.

The two-argument-longer form exists rather than a default because the atlas offset is zero for the overwhelming majority of calls and the original's authors were counting instructions in this loop. A rebuild has no reason to keep two entry points.
