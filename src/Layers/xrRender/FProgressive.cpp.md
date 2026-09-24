# src/Layers/xrRender/FProgressive.cpp

> Level of detail without swapping meshes: the vertices are ordered most-important-first and the indices are grouped per level, so choosing a detail level is choosing a draw range.

**Needs** — [`FProgressive.h`](FProgressive.h.md) · [`xrCore/FMesh.hpp`](../../xrCore/FMesh.hpp.md) · [`R_Backend.h`](R_Backend.h.md)
**Used by** — reached through its declarations in [`FProgressive.h`](FProgressive.h.md); callers name that, not this file.
**Tier floor** — T1: the table is read as a byte image and drives a draw's base offsets.

## Purpose

A progressive mesh is one mesh authored so that its vertices are sorted by importance and its triangles are grouped into nested sets. Drawing the first *n* vertices and the corresponding triangle group gives a coherent simplification of the whole model. Level of detail is then a *range*, not a different mesh — no extra memory, no popping between separately authored versions, no second buffer bind.

This is the mechanism the shipped level geometry uses for anything large enough to need simplification.

## The slide-window table — frozen

```text
RECORD SlideWindow             # one detail level
  offset     : int             # where this level's triangles start in the index buffer
  num_verts  : int             # how many vertices from the buffer's base this level uses
  num_tris   : int             # how many triangles to draw

RECORD SlideWindowTable
  reserved : four 32-bit values      # sixteen bytes, unread
  count    : int
  windows  : list<SlideWindow>       # ordered from MOST detailed to least
```

**Invariants**

- **The vertex range always starts at the model's base.** A level uses the *first* `num_verts` vertices, never an arbitrary slice. That is the entire requirement the mesh simplifier had to satisfy when it ordered the vertices, and it is why a progressive mesh cannot be built from an arbitrary triangle list.
- **The index ranges are disjoint and ordered.** Each level's triangles are contiguous, starting at its offset. They are not nested ranges of one another; each level has its own complete index list, which is why the index buffer is larger than a single mesh's would be. The *vertices* are shared, which is where the saving is.
- **Index zero is the most detailed level.** The table runs downward.
- The sixteen reserved bytes at the head of the table are read and stored and never used by anything. They are unrecoverable — most likely alignment padding or a dropped header.

## `load(name, stream, flags)`

**Contract** — loads as a static model, then reads the slide-window table. The fast geometry, when present, carries its own table inside the fast-path chunk. Allocates both tables.

**Invariants** — The two tables are independent and may have different counts: the fast geometry was simplified separately, and a depth-only mesh tolerates more aggressive simplification than a shaded one.

## `render(command_list, level_of_detail, use_fast_geometry)`

**Contract** — selects a detail level from the caller's value and draws that range. Accounts to the static bucket.

```text
FUNCTION render(command_list, lod, use_fast)
  table = the fast table when fast geometry was asked for and exists, else the normal one

  IF lod >= 0
    level = round((1 - clamp(lod, 0, 1)) * (table.count - 1))
    remember it as the last level used
  ELSE
    level = the last level used

  window = table[level]
  bind the geometry
  draw window.num_tris triangles, using the first window.num_verts vertices,
       starting at the index base plus window.offset
```

**Invariants**

- **The level-of-detail value runs the opposite way to the table index.** One means maximum detail and maps to index zero; zero means minimum and maps to the last entry. The inversion is in this one line and nowhere else.
- **A negative value means "reuse the last level".** This is how a second pass over the same model — a shadow pass after a colour pass, say — draws the *same* simplification without recomputing it. It is essential: two passes at two detail levels produce a shadow that does not match its caster. The remembered level is per model record, so a duplicated model has its own.
- The remembered level is **not** updated on the fast path, which reads the fast table but leaves the normal path's memory alone. That is correct — the two tables are not interchangeable — but it means a fast-geometry draw followed by a negative-value normal draw uses a stale level. Nothing in the shipped renderer sequences them that way.
- The rounding is to nearest, not toward zero, so the mapping is symmetric across the range.

**Notes** — Where the level-of-detail value comes from is not decided here: the draw stream computes it from the model's projected screen area. See the chapter README.

## `copy(source)`

**Contract** — shares both tables with the source by pointer.

**Notes** — The normal table is copied **by value** (a record holding a pointer) and the fast one **by pointer**, and neither is reference counted. Both duplicates therefore alias the original's allocation and the first destruction frees it for everyone. It is the same defect as the fast-mesh sharing in [`FVisual.cpp`](FVisual.cpp.md), from the same cause: the code has no vocabulary for "this belongs to the shared asset, not the instance". A rebuild that separates asset from instance has the problem by construction.
