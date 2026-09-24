# src/Layers/xrRender/xrStripify.cpp

> Reorders a mesh's triangles and renumbers its vertices for the post-transform vertex cache, and measures whether the reorder was worth keeping.

**Needs** — [`xrStripify.h`](xrStripify.h.md) · [`NvTriStrip.h`](NvTriStrip.h.md) · [`VertexCache.h`](VertexCache.h.md)
**Used by** — [`xrStripify.h`](xrStripify.h.md)
**Tier floor** — T2: index arrays of a fixed 16-bit element width, rewritten in place; no device call and no file format.

## Purpose

Hardware transforms a vertex once and keeps the result in a small FIFO; a triangle whose
vertices are all still in that FIFO costs no transforms at all. A mesh as authored arrives
in whatever order its exporter emitted, which is usually the order the artist built it in
and has nothing to do with locality. This file is the engine's front door to the reordering
pass: one call that rewrites the index list into a cache-friendly order, and one call that
counts what that order would cost.

It is a separate file from the reordering machinery it calls because it holds the two
decisions the machinery does not: that the output is a **plain indexed triangle list** and
never a strip, and that the reorder is **applied only if it measurably wins**. The
stripifier itself is a vendored 2001-vintage tool ([`NvTriStrip.cpp`](NvTriStrip.cpp.md))
and knows neither.

## State

Stateless.

## `stripify(indices, perturb, cache_size, min_strip_length)`

**Contract** — takes a triangle list as a flat 16-bit index array and rewrites it in place
into cache-friendly order, renumbering the vertices as it goes. `perturb` is an output the
caller must **pre-size to the mesh's vertex count**: on return, the vertex that belongs in
new slot `i` is the one currently in old slot `perturb[i]`. `cache_size` is the transform
cache size to optimize against — the engine passes the running device's own reported size.
Allocates working copies of the index array, blocks for the duration, and is called at
model load or build time only, never in a frame.

**Invariants**

- The vertex data is **not** touched. Between this call returning and the caller applying
  the permutation, the index array and the vertex array disagree; a rebuild that hands the
  permutation back must either apply it here or make that window impossible.
- Exactly one output group comes back, it is a triangle list, and it holds the same number
  of indices as the input. The engine asserts all three. Strip topology is deliberately
  requested and then thrown away — see the note.
- `perturb` is a permutation: every old slot appears exactly once, so the caller can
  gather into a fresh array without a reverse lookup.
- Every index in the output is below the vertex count. The renumbering cannot invent a
  vertex, only rename one.

```text
FUNCTION stripify(indices, perturb, cache_size, min_strip_length)
  configure stripifier with cache_size, min_strip_length, and lists-only

  groups = generate_strips(indices)            # reorders triangles for locality
  ASSERT groups has exactly one triangle-list group of the same index count

  renumbered = remap_indices(groups, count of perturb)
                                               # vertices renumbered in first-use order

  # The remap tells us, per index slot, the old and the new name of the same vertex.
  # Reading both arrays in lockstep recovers the whole permutation.
  FOR EACH position IN 0 .. count of indices - 1
    old_slot = groups[0].indices[position]
    new_slot = renumbered[0].indices[position]
    perturb[new_slot] = old_slot

  indices = renumbered[0].indices
```

**Notes** — Asking for strips and then demanding lists-only is not waste: the strip search
*is* the locality heuristic. Strips are built because a strip is by construction a run of
triangles that share vertices, and are then flattened back into a list because the engine
issues indexed list draws exclusively — one topology for every mesh, no per-mesh branch at
draw time. A rebuild is free to use any cache optimizer that emits a list; the strips are
an implementation detail of this one.

The renumbering is the half that transfers least obviously and matters most. Reordering
triangles alone improves the transform cache; renumbering vertices into first-use order
additionally makes the vertex buffer read front-to-back, which is what the memory system
wants. The two are separate passes in the tool and both are needed.

`min_strip_length` reaches the caller as zero, because lists-only discards the distinction
between a long strip and a short one. It survives as a parameter because the tool takes one.

## `simulate(indices, cache_size)`

**Contract** — returns how many vertex transforms the given index order would cost on a
device with a transform cache of `cache_size` entries. Pure: reads the index list, touches
nothing. Cheap enough to run twice around a reorder.

```text
FUNCTION simulate(indices, cache_size) -> int
  cache = empty FIFO of capacity cache_size
  transforms = 0
  FOR EACH id IN indices
    IF cache contains id   CONTINUE          # a hit costs nothing
    transforms = transforms + 1
    push id onto the front of cache, evicting the oldest
  RETURN transforms
```

**Notes** — This is the accept/reject oracle for the whole optimization, and that is the
decision a rebuild must keep. The caller measures the authored order, reorders, measures
again, and **keeps the new order only if the count went down**; otherwise it discards the
result and leaves the mesh as authored. The reordering heuristic is not monotone — a mesh
that is already well ordered, or one small enough to fit the cache entirely, can come back
worse — and there is no cheaper way to know than to simulate.

The cache model is a strict first-in-first-out queue of fixed capacity with no
associativity and no replacement policy, which is what the hardware of the era actually
did. A rebuild targeting modern hardware may find the real cache shallower or deeper, and
the only thing that changes is the number passed in.

## Who calls this

Exactly one caller: the detail-object model, which optimizes each of its small meshes once,
at load ([`DetailModel.cpp`](DetailModel.cpp.md)). Level geometry and character models are
*not* reordered at run time — they were optimized by the tools that built the shipped data,
and re-optimizing them would cost load time for nothing.

The call site is compiled only into the Direct3D build and only outside the editor, so on
the OpenGL backend detail meshes are drawn in their authored order and this whole file is
dead code. Nothing in the algorithm is backend-specific; the guard reads as an oversight
from when the OpenGL backend was added, and a rebuild should run the optimization on every
backend or on none.
