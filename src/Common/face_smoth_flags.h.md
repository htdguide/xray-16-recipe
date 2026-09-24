# src/Common/face_smoth_flags.h

> Unpacks the per-triangle bits that say which of its edges are smooth and whether the triangle is wound backwards, and decides from them whether two triangles may share a smoothed normal.

**Needs** — _(none)_
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: it reads bit positions out of a field whose layout is fixed by the authored mesh data.

## Purpose

A mesh arrives with, per triangle, a small bit field describing its edges. Three of the
bits say whether each edge is *hard* — a crease the lighting must not smooth across — and
one says the triangle is wound backwards relative to the surface it belongs to. This file
is the only place those bit positions are interpreted, and it exists so that the mesh
tangent generator and the level compiler agree on what a shared edge means.

The whole file is one decision: **a backwards-wound triangle has its edges renumbered**, so
that an edge index means the same edge to both triangles that share it.

## State

```text
RECORD FaceFlags
  bits : int (32-bit)
    # bit 0 : edge 0 is HARD          (set = hard, clear = soft — note the inversion)
    # bit 1 : edge 1 is HARD
    # bit 2 : edge 2 is HARD
    # bit 3 : the triangle is wound backwards
```

**Invariants**

- An edge index is always 0, 1 or 2 — checked in debug builds at every entry point.
- The sense of the edge bits is **inverted**: a *set* bit means the edge is hard, and the
  soft test negates. Nothing in the source explains the inversion; it is most likely that
  the field originally held a smoothing-group mask, where a set bit excluded the edge.
  Either way the encoding is frozen by authored data.
- The backface bit sits at position 3, immediately above the three edge bits. Bits above
  it are not touched here and may carry other meaning.

## Edge renumbering

**Contract** — given the flags and an edge index in a triangle's own winding, yield the index of
the same physical edge in the canonical winding.

```text
FUNCTION canonical_edge(flags, edge_index) -> int
  IF NOT backface(flags)
    RETURN edge_index
  RETURN (4 - edge_index) MOD 3     # 0->1, 1->0, 2->2
```

**Notes** — the mapping swaps edges 0 and 1 and fixes edge 2. That is what reversing a
triangle's vertex order does to its edge numbering, and the arithmetic form is just a
compact way to write the swap.

## `is_backface` / `set_backface`

**Contract** — read and write the winding bit. Setting it does not renumber anything already
stored; it only changes how subsequent edge queries are interpreted.

## `is_soft_edge` / `set_soft_edge`

**Contract** — read and write the hardness of one edge, addressed in the triangle's own winding.
Both renumber first, so a caller never has to think about winding.

```text
FUNCTION is_soft_edge(flags, edge_index) -> bool
  i <- canonical_edge(flags, edge_index)
  RETURN bit i of flags is clear          # clear means SOFT

FUNCTION set_soft_edge(flags, edge_index, soft)
  i <- canonical_edge(flags, edge_index)
  IF soft  THEN clear bit i  ELSE set bit i
```

## `may_smooth_across`

**Contract** — given two triangles that share an edge, and that edge's index in each of their
own windings, decide whether lighting may be smoothed across it. This is the question the
tangent generator and the normal averaging pass actually ask.

```text
FUNCTION may_smooth_across(flags_a, flags_b, edge_in_a, edge_in_b) -> bool
  IF backface(flags_a) != backface(flags_b)
    RETURN false                    # opposite-facing neighbours are two surfaces,
                                    # not one surface with a crease
  RETURN is_soft_edge(flags_a, edge_in_a)
     AND is_soft_edge(flags_b, edge_in_b)
```

**Invariants** — softness must be agreed by *both* triangles. One side declaring a crease is
enough to produce one, which is what makes the authored data's intent survive a mesh
operation that only touched one triangle.

**Notes** — the winding check is the interesting half. Two triangles wound oppositely across a
shared edge are back-to-back faces of a thin surface, not two faces of one solid, and
smoothing between them would average a normal with its own opposite. This is exactly the
case [`NvMender2003/NVMeshMender.cpp`](NvMender2003/NVMeshMender.cpp.md) handles
differently — it decides smoothing by the angle between face normals, which rejects
back-to-back faces as a side effect rather than by rule.
