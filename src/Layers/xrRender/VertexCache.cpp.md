# src/Layers/xrRender/VertexCache.cpp

> A model of the graphics hardware's post-transform vertex cache, used to score candidate triangle orderings while a mesh is being stripified.

**Needs** — [`VertexCache.h`](VertexCache.h.md)
**Used by** — reached through its declarations in [`VertexCache.h`](VertexCache.h.md); callers name that, not this file.
**Tier floor** — T3: it is a fixed-size most-recently-used list of integers. Nothing here touches a device; it only *predicts* one.

## Purpose

When the engine reorders a mesh's triangles for drawing, the thing it is optimizing is how often a vertex it needs is still sitting in the hardware's small cache of already-transformed vertices. It cannot ask the hardware, so it simulates it: a fixed-capacity list, most-recent first, into which every referenced vertex index is pushed. The stripifier tries an ordering, replays it through this model, counts the hits, and keeps the best.

The model is deliberately crude — a strict most-recently-used list of a fixed size, with no notion of which hardware is actually present. That is the right level of fidelity: the ordering that wins on a 16-entry model wins on almost any real cache, and the alternative is per-device tuning of data that ships precomputed.

## State

```text
RECORD VertexCache
  entries : list<int>       # fixed length, index 0 is most recent; -1 means empty
```

**Invariants** — the length never changes after construction; the default is **16**, which is the post-transform cache size of the hardware generation this engine was written for and is the number the shipped meshes were optimized against. Vacancy is spelled as the sentinel `-1` rather than a separate count, so a vertex index is never negative.

## `VertexCache(size)` / `VertexCache()`

**Contract** — builds a cache of the given capacity, all slots empty. The no-argument form is the 16-entry default.

## `in_cache(index) -> bool`

**Contract** — whether the vertex is currently modelled as resident. A linear scan: at sixteen entries that beats any indexed structure, and the scan is the innermost operation of the stripifier's scoring loop.

## `add(index) -> int`

**Contract** — pushes a vertex to the most-recent position, shifting everything else one slot older, and returns the index evicted off the end. The evicted value is what the caller needs to know; it is not a status.

```text
FUNCTION add(index) -> int
  evicted := entries[last]
  shift entries right by one, dropping the last
  entries[0] := index
  RETURN evicted
```

**Notes** — Every reference is treated as a miss-then-insert: an index already resident is pushed to the front again rather than left where it was, which means a run of the same index keeps it hot. Real hardware behaves the same way for the purpose this is used for, and the scoring loop asks `in_cache` *before* calling this, so the hit is counted separately.

## `clear()` / `copy_into(other)` / `at(i)` / `set(i, value)`

**Contract** — `clear` marks every slot empty; `copy_into` writes this cache's contents over another's, which is how the stripifier snapshots a state, tries a branch and restores it; the indexed pair is the raw access the snapshot and the scoring loop use. `copy_into` copies as many entries as *this* cache holds, so the two must be the same size — a mismatch is an authoring error in the stripifier, not a runtime condition.
