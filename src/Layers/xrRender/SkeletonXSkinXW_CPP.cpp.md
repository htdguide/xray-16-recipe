# src/Layers/xrRender/SkeletonXSkinXW_CPP.cpp

> Software skinning, portable form: for each vertex, blend the position and normal produced by up to four bone transforms, weighted, and write the result straight into device-mapped memory.

**Needs** — [`SkeletonXSkinXW.h`](SkeletonXSkinXW.h.md) · [`SkeletonXVertRender.h`](SkeletonXVertRender.h.md) · [`xrCore/Animation/Bone.hpp`](../../xrCore/Animation/Bone.hpp.md) · [`xrCore/Threading/ParallelFor.hpp`](../../xrCore/Threading/ParallelFor.hpp.md)
**Used by** — [`SkeletonXSkinXW.h`](SkeletonXSkinXW.h.md) · [`SkeletonXSkinXW_SSE.cpp`](SkeletonXSkinXW_SSE.cpp.md)
**Tier floor** — T1: it writes a fixed byte layout into a mapped device buffer, and the loop is on the frame's critical path.

## Purpose

This is the reference implementation of the four skinning routines, and the one that runs everywhere except one legacy target. It is also the **definition** of what skinning means in this engine — the 4-wide file is required to produce the same vertices, so any disagreement between the two is a bug in that one, not a choice.

The split into two files is **not load-bearing**. One algorithm is written twice, once portably and once against a specific instruction set; a rebuild writes it once in whatever 4-wide form its language offers and deletes the other file. What must survive is the algorithm below and the numerical rules attached to it.

## State

`Stateless.`

## The blend

**Contract** — each routine reads a run of source vertices and writes the same number of output vertices. It reads the bone instances' *render* transforms — the model-base-to-bone-to-model matrices, not the local-to-model ones — and touches nothing else. It allocates nothing. The destination is device-mapped memory, so it is written once, forward, and never read back.

```text
FUNCTION skin(destination, source, count, bones)
  FOR EACH vertex i IN 0 .. count-1
    accumulated_position = zero
    accumulated_normal   = zero
    running_weight       = 1
    FOR EACH influence j IN 0 .. (influences - 2)
      M = bones[source[i].bone[j]].render_transform
      accumulated_position = accumulated_position + M applied to source[i].position * weight[j]
      accumulated_normal   = accumulated_normal   + M rotated  source[i].normal   * weight[j]
      running_weight = running_weight - weight[j]
    M = bones[source[i].bone[last]].render_transform      # the implied last influence
    accumulated_position = accumulated_position + M applied to source[i].position * running_weight
    accumulated_normal   = accumulated_normal   + M rotated  source[i].normal   * running_weight
    destination[i] = (accumulated_position, accumulated_normal, source[i].u, source[i].v)
```

**Invariants**

- **The last weight is computed, never read.** It is one minus the sum of the stored weights. The source data guarantees the stored weights sum to at most one; nothing checks it, and a mesh that violates it produces a negative contribution rather than a diagnostic.
- **Normals are transformed by the rotation part only** — the translation is dropped, the matrix is otherwise applied in full. No inverse-transpose is used, which is correct only because bone transforms are rigid (rotation and translation, no shear or non-uniform scale). A rebuild that allows scaled bones must change this and will then not match the original.
- **Normals are not renormalized.** A blend of two unit vectors is shorter than one, most visibly on a vertex split evenly between two bones at a sharp angle. The engine accepts the shortening — the shader that consumes these vertices normalizes, or the lighting error is small enough not to matter. A rebuild that normalizes here produces slightly different, arguably better, shading and will not match pixel for pixel.
- **Texture coordinates are copied verbatim** and the tangent frame is dropped entirely; see [`SkeletonXVertRender.h`](SkeletonXVertRender.h.md).
- The four routines must agree with the per-vertex helpers used by the picking and decal paths in [`SkeletonX.cpp`](SkeletonX.cpp.md) — they compute the same blend for position alone, and a divergence would put a decal off the surface it was stamped on.

## Per-influence-count differences

Each count is written as its own routine rather than one parameterized loop, and the four differ in more than their arithmetic:

- **One influence** — no blend at all: transform by the single bone. The loop is unrolled eight-fold with a scalar remainder pass. The unrolling is an artifact of the compilers of the era; it buys nothing now and a rebuild should delete it.
- **Two influences** — a linear interpolation between the two transformed results rather than an accumulation of weighted terms. Mathematically identical, different rounding. It also carries a **special case**: when both bone references are the same bone, the second transform is skipped entirely. Meshes authored with a uniform two-influence format contain many such vertices, and the branch is worth its cost.
- **Three influences** — the accumulating form, no special case, no unrolling.
- **Four influences** — the accumulating form, and the only one of the four that is **run across worker threads**, split into ranges over the vertex array. Four-influence meshes are the largest in the shipped data, and the work is trivially parallel because each output vertex depends on one input vertex.

**Notes** — That only the four-influence routine is parallelized is a pragmatic choice, not a constraint: the others are equally parallel. A rebuild should decide by vertex count rather than by influence count, and must keep the property that makes it safe — the destination ranges are disjoint, and the bone instances are read-only for the duration of the call.

## Could not recover

Why the two-influence routine interpolates while the three- and four-influence routines accumulate. The results agree to rounding, so nothing depends on it; the likeliest reading is that the two-influence path was written first, against a vector type that had an interpolation operation and no weighted accumulate.
