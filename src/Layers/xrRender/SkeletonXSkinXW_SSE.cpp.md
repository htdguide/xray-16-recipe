# src/Layers/xrRender/SkeletonXSkinXW_SSE.cpp

> The same four skinning routines as the portable file, expressed as 4-wide float work — and the byte-layout assumptions that expression makes.

**Needs** — [`SkeletonXSkinXW.h`](SkeletonXSkinXW.h.md) · [`SkeletonXSkinXW_CPP.cpp`](SkeletonXSkinXW_CPP.cpp.md) · [`SkeletonXVertRender.h`](SkeletonXVertRender.h.md) · [`xrCore/Animation/Bone.hpp`](../../xrCore/Animation/Bone.hpp.md)
**Used by** — [`SkeletonXSkinXW.h`](SkeletonXSkinXW.h.md)
**Tier floor** — T1, and lower than the rest of the chapter in practice: it addresses the bone-instance array by a hard-coded byte stride and the transform by a hard-coded byte offset.

## Purpose

One algorithm, second spelling. The blend, the implied last weight, the rigid-transform assumption and the decision not to renormalize are all defined in [`SkeletonXSkinXW_CPP.cpp`](SkeletonXSkinXW_CPP.cpp.md) and are not restated here; this page records only what the 4-wide expression adds, because that is the part a rebuilder must either reproduce or deliberately drop.

**The two-file split is not load-bearing.** A rebuild writes the blend once in its language's 4-wide form and deletes both files. This one exists only for a single legacy 32-bit target, is compiled out everywhere else, and is written in an assembly dialect that no longer has a portable equivalent — which is precisely why it should not be transcribed.

## State

`Stateless.`

## What the 4-wide form decides

### The lane layout

A three-component vector occupies the low three lanes of a four-lane register with the fourth ignored; a transform is held as three registers, each a broadcast of one source component, multiplied against three consecutive rows of the bone matrix and summed. Applying a transform to a *direction* is those three multiply-adds; applying it to a *position* is the same plus the matrix's fourth row. The accumulation order is therefore fixed: contribution of bone 0, then 1, then 2, then 3, matching the portable file exactly — which is what keeps the two producing the same floats.

### The output is emitted as two stores, not four

The output record is thirty-two bytes, which is exactly two four-lane registers. The routine shuffles the finished position and normal so that the first register holds the three position components plus the normal's first component, and the second holds the normal's remaining two components plus the texture coordinate pair. This is the reason for the record's field order and its two-byte packing ([`SkeletonXVertRender.h`](SkeletonXVertRender.h.md)) — a different field order would cost extra shuffles or extra stores per vertex.

Both stores are **non-temporal**: the deformed vertices are written to device-mapped memory and will never be read back by the processor, so nothing is gained by keeping them resident in cache and a great deal is lost by evicting the source data to make room. The routine ends by fencing the stores so the device sees them. A rebuild whose language offers a non-temporal store should use it here; one that does not will pay a measurable cache cost on large meshes and nothing else.

The source stream is prefetched a fixed distance ahead — the source vertices are walked strictly forward and the hardware prefetcher of the era did not reliably cover the 60-to-76-byte stride.

### The bone-instance addressing — the fragile part

The bone matrix is reached by **computing a byte offset**: the bone index is multiplied by the size of one bone instance, and the transform's rows are read at fixed offsets within it.

```text
byte address of bone b's render transform row r
  = bone_array_base + b * 160 + 64 + r * 16
```

**Invariants**

- One bone instance is **exactly 160 bytes**, and its render transform begins at byte **64** — after the local transform, which occupies the first 64. Both numbers are baked into the address arithmetic. They hold only for the 32-bit build this file is compiled for; the record's callback pointers are wider at 64 bits, which changes the stride.
- The *render* transform is the one read, not the local one at offset zero. Picking the wrong sixty-four bytes yields a plausible-looking but wrong pose.
- The constant 1.0 needed for the implied last weight is materialized from its bit pattern rather than loaded from memory.

This addressing is the reason the file is confined to one architecture and one pointer width, and it is the single strongest argument for the recipe's position that the split is incidental: a rebuild that keeps a field-addressed record and lets the compiler choose the offsets gets the same speed today and none of the fragility.

## Per-routine differences

Same as the portable file: the one-influence routine has no blend, the two-influence routine interpolates rather than accumulates, and the three- and four-influence routines accumulate weighted terms with the last weight derived. This file does **not** thread the four-influence routine; the portable file does. On the one target where this file is used, that is a real behavioural difference in throughput and none in output.

## Could not recover

Whether this path was ever measured against the portable one on a compiler younger than the code. The file is kept for a target that the project's own platform list barely reaches, and there is no record of the margin it was written to win.
