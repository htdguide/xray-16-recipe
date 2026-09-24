# src/Common/NvMender2003/mender_input_output.h

> Marshals any of the engine's mesh representations into the tangent generator's flat vertex-and-index form, and reassembles the result — including the per-vertex data the generator does not carry.

**Needs** — [`NVMeshMender.h`](NVMeshMender.h.md) · [`convert.h`](convert.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: it is a data reshaping pass with no layout or timing constraint.

## Purpose

The tangent generator understands exactly one vertex shape: position, texture coordinate,
normal, tangent, binormal. The engine has several mesh representations with more per-vertex
data than that — colours, bone weights, a second texture coordinate. This file is the
bridge, and the interesting half is the *return* trip, because the generator can add
vertices and the extra data has to follow them.

## State

Stateless.

## Caller obligations

**Contract** — this bridge is generic over the caller's vertex and face types, and asks the
caller to supply three operations for them:

```text
FUNCTION face_vertex(face, corner_index) -> vertex index     # readable and writable
FUNCTION set_vertex(generator_vertex, source_vertex)          # source -> generator form
FUNCTION set_vertex(destination_vertex, original_vertex, generator_vertex)
                                                              # merge back: take the
                                                              # generator's computed basis,
                                                              # keep the original's extras
```

**Invariants** — the three-argument merge is the one that matters. It receives *both* the
generator's result and the original vertex the result descends from, and it decides field by
field which wins. Getting this wrong loses per-vertex colour or skinning weights on exactly
the vertices the generator duplicated — a subtle corruption confined to creased edges.

## `fill_mender_input`

**Contract** — flatten a mesh into the generator's two arrays. Vertices are converted one for
one in order; faces contribute their three corner indices in order. Both output arrays are
emptied first.

```text
FUNCTION fill_mender_input(vertices, faces) -> (mender_vertices, indices)
  mender_vertices <- empty, sized to match vertices
  FOR EACH v AT i IN vertices
    set_vertex(mender_vertices[i], v)
  indices <- empty
  FOR EACH f IN faces
    append face_vertex(f, 0), face_vertex(f, 1), face_vertex(f, 2) TO indices
```

**Invariants** — vertex index `i` in the input is vertex index `i` in the generator's array.
The whole return trip depends on that identity.

## `retrieve_data_from_mender_output`

**Contract** — write the generator's result back over the caller's mesh. The face list keeps its
length and order and only its indices change; the vertex list is *replaced*, because the
generator may have added vertices.

```text
FUNCTION retrieve_data_from_mender_output(vertices, faces,
                                          mender_vertices, indices, new_to_old)
  originals <- a copy of vertices          # taken BEFORE vertices is replaced
  FOR EACH face AT i IN faces
    set_face(face, indices[3i], indices[3i + 1], indices[3i + 2])
  vertices <- a new list, one entry per mender vertex
  FOR EACH j IN 0 .. count(mender_vertices) - 1
    set_vertex(vertices[j], originals[new_to_old[j]], mender_vertices[j])
```

**Invariants**

- The copy of the original vertices must be taken before the destination is cleared. It is
  the only surviving source of the per-vertex data the generator does not carry.
- The face count never changes: the generator splits vertices, never triangles. The
  assertion that indices are exactly three times the face count is what catches a mismatched
  pairing of this function with the fill above.
- `new_to_old` is the generator's mapping and is indexed by *new* vertex index. For a
  vertex the generator did not touch it is the identity.

**Notes** — the copy of every original vertex is a real allocation proportional to the mesh,
taken per call. This runs at asset-build time, not per frame, so the cost is paid once; a
rebuild processing meshes at load time should note it.
