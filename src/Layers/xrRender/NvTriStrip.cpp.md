# src/Layers/xrRender/NvTriStrip.cpp

> The front door of the stripifier: four tuning knobs, one entry point that turns a triangle list into cache-ordered primitive groups, and a second that renumbers vertices so the vertex buffer is walked front-to-back.

**Needs** — [`NvTriStrip.h`](NvTriStrip.h.md) · [`NvTriStripObjects.h`](NvTriStripObjects.h.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`NvTriStrip.h`](NvTriStrip.h.md)
**Tier floor** — T2: no device contact and no file format, only index arithmetic. What stops T3 is the cost — the passes underneath are quadratic in triangle count and run at level load on meshes of tens of thousands of triangles, which wants flat arrays and compiled throughput.

## Purpose

A post-transform vertex cache sits between the vertex shader and the rasterizer: a vertex whose index was seen recently is reused instead of re-shaded. The *order* of indices in the index buffer therefore decides how much of the mesh is shaded twice. This file is the entry point of a mesh reordering tool that makes that order good — it groups triangles into strips that run along the surface, then orders the strips so that consecutive strips share vertices.

It is a compile-time-ish tool, not a frame-time one: it runs once when a mesh is prepared, and its output is an index buffer permutation plus (optionally) a vertex permutation. The substance of the algorithm lives in [`NvTriStripObjects.cpp`](NvTriStripObjects.cpp.md); this file is the interface, the output packaging, and the vertex renumbering.

The split between the two files is the original library's and is close to arbitrary — this is the public surface, that is the machinery. A rebuild may merge them.

## State

Four module-level settings, read at the next call and not otherwise:

```text
RECORD StripifierSettings          # one global set, mutated between calls
  cache_size     : int, default 16   # the device's post-transform cache, in vertices
  stitch_strips  : bool, default true
  min_strip_size : int, default 0    # in triangles; shorter strips are demoted to a list
  lists_only     : bool, default false
```

**Invariants**

- These are *settings*, not arguments, so two meshes cannot be stripified concurrently. The machinery below has the same property (it keeps scratch state across calls). A rebuild should pass a settings record by value and lose the restriction for free; nothing depends on the globals being global.
- `cache_size` is the *actual* hardware cache size, not a derated one. The derating happens one level down — see [`NvTriStripObjects.cpp`](NvTriStripObjects.cpp.md), which subtracts a fixed inefficiency margin. Two sizes are named in the interface because two generations of hardware were the targets when this was written: 16 and 24. The engine passes the running device's reported cache size.
- Indices are 16-bit throughout, in and out. A mesh handed to this tool must therefore have at most 65 536 vertices. Nothing checks it; the renumbering silently wraps if it is violated.

```text
RECORD PrimitiveGroup
  type        : enum { LIST, STRIP, FAN }
  num_indices : int
  indices     : list<int (16-bit)>    # owned by the group; released with it
```

`FAN` is declared and never produced. Fans were the third primitive topology of the era; the stripifier never emits one.

## `GenerateStrips`

**Contract** — takes a flat triangle list (index count a multiple of three) and returns a sequence of primitive groups whose concatenated triangles are exactly the input's, reordered. Allocates each group's index array; ownership passes to the caller. Does not touch vertex data, does not block, is not reentrant. Input index order is irrelevant to correctness and relevant only to tie-breaking.

**Invariants**

- Every input triangle appears exactly once in the output, with its winding preserved. This is not decoration: the renderer culls back faces, so a triangle emitted with reversed winding disappears.
- The output *shape* depends on the settings, and is one of exactly three:
  - **lists-only** — exactly one `LIST` group holding every index. The strips were still computed; they were then flattened back to triples. The product is the *ordering*, not the topology.
  - **stitched strips** — exactly one `STRIP` group. The engine asserts this, because the whole point of stitching is that one draw call covers the mesh.
  - **separate strips** — one `STRIP` group per strip, followed, only if any triangles were left over, by exactly one `LIST` group as the final group. The caller may rely on the leftover list being last.

```text
FUNCTION generate_strips(indices, settings) -> list<PrimitiveGroup>
  strips, leftover_faces <- stripify(indices, settings.cache_size,
                                     settings.min_strip_size)
      # strips are in cache-optimal order; leftover_faces are the triangles
      # from strips too short to be worth keeping, themselves cache-ordered

  IF settings.lists_only
    # flatten everything back to triples, strips first, leftovers last.
    # The strip structure was a means of finding a good order, and is discarded.
    RETURN [ LIST group of (every strip's faces, then every leftover face) ]

  flat, separate_count <- create_strips(strips, settings.stitch_strips)
      # flat is one index stream; when not stitching, a sentinel value
      # separates consecutive strips

  groups <- empty
  FOR EACH strip span IN flat            # split at the sentinel; when stitching
                                          # there is exactly one span
    groups.append(STRIP group of that span)
  IF leftover_faces is not empty
    groups.append(LIST group of leftover_faces)     # always last
  RETURN groups
```

**Notes**

- The sentinel is a reserved index value that cannot occur in real data; the scan for it advances past one extra position per strip. A rebuild with a list-of-lists return type deletes the sentinel and the scan together — nothing else depends on the flat encoding.
- The tool keeps its own working copy of the triangles as mutable face records with mark bits, and frees them here. That storage is scratch for one call; the only thing that survives is the index arrays.
- In the shipped engine only the **lists-only** path runs: the one caller sets lists-only, a minimum strip size of zero, and never touches stitching. The strip and stitch machinery is legacy from the library this file came from. A rebuild that only needs to match shipped behaviour can implement lists-only and drop the rest — but then it is implementing a *cache-order optimizer*, and should say so, because "stripifier" stops being the honest name.

## `RemapIndices`

**Contract** — renumbers vertices into first-use order. Given the primitive groups and the vertex count, returns a parallel set of groups whose indices are the new numbering. Allocates the new index arrays; ownership passes to the caller. Does not reorder triangles — group order, group types and index positions are untouched, only the values change.

**Invariants**

- The result is a bijection on the vertices that are actually referenced. A vertex referenced by no triangle gets no new number and is therefore *dropped* from the numbering, which silently shortens the buffer the caller is expected to build. The engine's meshes reference every vertex, so this never bites; a rebuild that cannot assume that must detect it.
- After the caller applies the permutation to the vertex buffer, the index stream has the property that the first appearance of new index *n* precedes the first appearance of *n+1*. The vertex buffer is then read essentially front-to-back, which is what makes the pre-transform fetch (a straight memory read, not a cache) sequential.

```text
FUNCTION remap_indices(groups, vertex_count) -> list<PrimitiveGroup>
  new_number <- map of old index -> new index, initially empty
  next <- 0
  FOR EACH group IN groups                 # group order is the draw order,
    FOR EACH old IN group.indices          # so first use means first drawn
      IF old not in new_number
        new_number[old] <- next
        next <- next + 1
      emit new_number[old]
```

**Notes**

- This hands back the *forward* map — old positions renumbered. The caller that wants to permute its vertex buffer needs the inverse, and builds it by walking the two index streams in lockstep: for each position, the new index it now carries names the slot that must receive the vertex the old index named. Getting that direction backwards produces a mesh that is subtly scrambled rather than obviously broken, which is why the one caller does it in a separate, explicit pass.
- The old-to-new map is a dense array sized by the vertex count and pre-filled with a "not seen" marker, not a hash table. At these sizes — thousands of vertices — the dense array is both simpler and faster, and the vertex count is already known because the index width caps it.
- The vertex count parameter is 16-bit, which is the tool's way of restating the index-width limit. A rebuild should make the width explicit rather than inferring it from a parameter type.

## `SetCacheSize`, `SetStitchStrips`, `SetMinStripSize`, `SetListsOnly`

**Contract** — each assigns one field of the settings record above; no validation, no side effects, effective from the next `GenerateStrips`. They exist as separate calls rather than a settings argument because the original library's interface was C-shaped. A rebuild should replace all four with one argument.
