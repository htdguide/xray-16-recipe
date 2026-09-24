# src/Layers/xrRender/FVisual.cpp

> A static model: bind a slice of the level's shared buffers and draw it — or draw its position-only twin when the pass only needs depth.

**Needs** — [`FVisual.h`](FVisual.h.md) · [`BufferUtils.h`](BufferUtils.h.md) · [`xrCore/FMesh.hpp`](../../xrCore/FMesh.hpp.md) · [`R_Backend.h`](R_Backend.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: it maps and fills device buffers and issues draws with explicit base offsets.

## Purpose

The simplest and most numerous model type. Its whole substance is the *four different ways a model's geometry can arrive*, and the fast-geometry idea.

## The four geometry sources

```text
FUNCTION load(name, stream, flags)
  read the shared model header                  # see FBasicVisual.cpp

  IF the stream has a GEOMETRY-CONTAINER chunk        # a LEVEL model
    read (buffer id, base vertex, vertex count) and (buffer id, base index, index count)
    take a reference on each of the level's shared buffers
    take the vertex layout that was declared with the vertex buffer
    IF a FAST-PATH chunk is also present and the device supports it
      read a second container the same way, out of the SEPARATE fast buffers
  ELSE                                                # a STANDALONE model
    IF a container chunk of either kind is present
      (same as above — but this path is dead; see below)
    ELSE
      read an inline vertex block: a packed layout descriptor, a count, and the bytes
      build a private vertex buffer and upload them
      read an inline index block: a count and the bytes
      build a private index buffer, marked readable, and upload them

  IF the caller asked for no vertices, stop here
  declare the geometry from whichever layout was found
```

**Invariants**

- **A level model owns nothing.** Its geometry is a (buffer, base, count) triple into buffers the level loader built once — see the resource manager's geometry container. This is what lets a level of a hundred thousand triangles draw with a handful of buffer binds. A rebuild must keep it; per-model buffers cost a bind per draw.
- **A standalone model owns its buffers.** A weapon, a creature, an item — anything loaded outside a level — builds its own. Its index buffer is created **readable**, because the decal system needs to read a model's triangles back to cut a wallmark into them. Its vertex buffer is not.
- Two of the four paths — a standalone model naming a container — are **unreachable** and assert immediately if taken, with a note asking whoever hit it to report it. They are the remains of an earlier arrangement in which standalone models also shared buffers. A rebuild deletes them.
- A model may be loaded with vertices suppressed. Only the structure and the bounds come back. This is used where a model is wanted as a *shape* rather than as something to draw.

## Fast geometry

**Contract** — an optional second mesh, authored into the same file, holding the same shape with a minimal vertex layout: position only, no normals, no texture coordinates, no tangent frame.

**Invariants**

- It is used for passes that write only depth — shadow maps and the depth pre-pass — where everything but position is dead weight. The saving is bandwidth, not triangles: the same geometry at a third of the vertex size.
- It lives in **separate shared buffers** from the ordinary geometry. That is what makes it worth having: a shadow pass binds the fast buffers once and never touches the full ones, so the two layouts never interleave.
- It is used only when the caller asks for it *and* the device is not running hardware tessellation. Tessellation needs the full vertex attributes to displace with, so the fast path is silently declined. That interaction is the one non-obvious condition on the whole feature.
- On the oldest renderer generation it is gated behind a console setting instead, and off by default. It was added for the newer ones.

## `render(command_list, level_of_detail, use_fast_geometry)`

**Contract** — binds the geometry and issues one indexed triangle-list draw. Accounts the vertices to the *static* bucket of the draw statistics. The level-of-detail argument is ignored — a static model has one level; see [`FProgressive.cpp`](FProgressive.cpp.md) for the type that uses it.

## `copy(source)`

**Contract** — shares the source's geometry: the same declaration, the same buffers with their references incremented, the same bases and counts, and the same fast mesh.

**Invariants** — The fast mesh is copied **as a pointer, without a reference count**, while the buffers underneath it are counted. The duplicate and the original therefore share one fast-mesh record, and whichever is destroyed first destroys it for both. This is a real defect in the original — a duplicated static model with fast geometry that outlives its source has a dangling mesh record. It is not observed in practice because the model pool destroys duplicates before originals, but a rebuild must not reproduce the arrangement: the fast mesh is part of the shared *asset*, not the per-instance record, and belongs on whatever object owns the asset.
