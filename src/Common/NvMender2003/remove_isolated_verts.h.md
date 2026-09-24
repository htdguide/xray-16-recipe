# src/Common/NvMender2003/remove_isolated_verts.h

> Rebuilds a mesh containing only the vertices some triangle actually references, renumbering the triangles as it goes.

**Needs** — _(none)_
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: an index remapping pass over arrays; no layout or timing constraint.

## Purpose

Mesh processing leaves orphans. Splitting a mesh by material, discarding degenerate
triangles or extracting a level-of-detail all leave vertices that no remaining triangle
references, and those vertices still cost buffer space and still get skinned and
transformed. This pass removes them.

It also, as a side effect, **reorders the surviving vertices into triangle-visit order**,
which improves vertex-cache locality. Nothing in the source says that was intended, but it
is a real consequence and a rebuild that replaces this with a different compaction loses it.

## State

Stateless — a remap table is built and discarded per call.

## Caller obligations

**Contract** — generic over the caller's vertex and face types; requires the same corner accessor
the bridge in [`mender_input_output.h`](mender_input_output.h.md) asks for:

```text
FUNCTION face_vertex(face, corner_index) -> vertex index     # readable and writable
```

## `remove_isolated_verts`

**Contract** — given a vertex list and a face list, produce a new pair in which every vertex is
referenced by at least one face and every face refers to the new numbering. Faces are
neither added, removed nor reordered. Allocates the new lists and one remap table the size
of the input vertex list.

```text
FUNCTION remove_isolated_verts(vertices, faces) -> (new_vertices, new_faces)
  remap <- a table of size count(vertices), every entry UNASSIGNED
  new_vertices <- empty
  new_faces    <- empty
  FOR EACH face IN faces                     # in order: this is what fixes the new ordering
    new_face <- empty
    FOR corner IN 0 .. 2
      old <- face_vertex(face, corner)
      IF remap[old] is UNASSIGNED            # first sighting: copy the vertex across now
        append vertices[old] TO new_vertices
        remap[old] <- index of the appended entry
      face_vertex(new_face, corner) <- remap[old]
    append new_face TO new_faces
```

**Invariants**

- A vertex is copied at its **first reference**, so the output order is the order triangles
  first mention vertices. That is the locality property above.
- The remap table's unassigned marker is the largest representable index value, which means
  a mesh may not legitimately contain that many vertices. At 32 bits this is four billion
  and not a practical limit; a rebuild should still prefer an explicit optional.
- Duplicate vertices are *not* merged. Two vertices with identical data both survive if both
  are referenced. Deduplication is a separate concern and this pass does not do it.

**Notes** — the in-place form of this operation copies both input lists, runs the pass, and
writes the results back over the originals. The copy is unavoidable given that the output
is built by appending, and the peak memory is therefore twice the mesh. A rebuild could
compact in place with a second pass at the cost of losing the reordering.

The two forms are declared with the same name and told apart by how many arguments they
take, which is a host-language convenience. They are two operations — *compact into* and
*compact in place* — and a rebuild should name them separately.
