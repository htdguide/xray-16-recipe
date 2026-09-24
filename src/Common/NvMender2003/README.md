# src/Common/NvMender2003 — the vendored tangent-basis generator

## What this module is responsible for

Computing a per-vertex **tangent basis** — a tangent, a binormal and a normal at every
vertex — for meshes that are lit per pixel from a normal map. Given positions, triangle
indices and texture coordinates, it produces the frame that maps the normal map's texture
directions onto the surface, and it splits vertices wherever one frame cannot honestly
serve two adjacent triangles.

This is a third-party library, dated 2003, adopted rather than written. Its allocator and
containers were swapped for the engine's and its one graphics-library dependency was
replaced with local arithmetic; otherwise it is the vendor's code and the vendor's
structure, including a defect and several stated limitations that the twins record.

Two further files in the directory are the engine's own: a bridge that marshals any of the
engine's mesh representations into the generator's flat form and back, and an index
compaction pass that is used alongside it.

## Where it sits

Part of [chapter 1](../README.md), with no dependency on anything outside this directory
except the portable graphics-enumeration header
[`d3d9compat.hpp`](../d3d9compat.hpp.md), which it uses only for a vector type and a
vertex-format constant.

It runs at **asset build time**, not per frame and not per level load. That is what
licenses its allocations, its per-position map and its inner loop over face pairs.

## The load-bearing ideas

### Smoothing is decided per position, not per vertex

A mesh usually holds several vertices at one position, duplicated because they differ in
texture coordinate. Building adjacency on vertex *indices* would see those as unrelated
surfaces and smooth nothing. The generator keys its adjacency on **position** instead,
which reunites them — and offers a mode that turns this off for callers whose existing
splits are meaningful.

Positions are compared **exactly**, never within a tolerance. A tolerance-based comparison
is not a valid ordering and a map built on one misplaces entries. The cost is that
positions differing in the last bit never smooth together; the fix, if a rebuild needs one,
is to quantize before keying, not to compare fuzzily.

### Three independent smoothing passes

The normal, the tangent and the binormal are each smoothed separately, with their own
threshold. A position may be split for one and not the others. The tangent and binormal
thresholds are what catch a *texture* seam that is not a geometric crease — two triangles
lying flat against each other but mapped from opposite sides of a mirrored texture.

### Splitting a vertex means the caller's data must follow it

The pass returns more vertices than it was given, so it also returns a mapping from each
new vertex to the original it descends from. The mapping always names an original, even for
a vertex split twice. Everything the caller knows about a vertex that the generator does
not carry — colour, skinning weights, a second texture coordinate — is recovered through
that mapping, which is the job of the bridge file.

The triangle count never changes. Only vertices are added and only corner indices rewritten.

### Unnormalized on purpose

Face normals, tangents and binormals are left unnormalized through the averaging, so their
magnitudes weight their contribution — by triangle area for normals (adjustable), by
texture-space stretch for the other two. Normalization happens once, at the end, in the
orthogonalization pass.

## The twins

| File | Role |
|---|---|
| [`NVMeshMender.cpp`](NVMeshMender.cpp.md) | The algorithm: smoothing groups, vertex splitting, the texture-gradient solve, orthogonalization, cylindrical seam repair |
| [`NVMeshMender.h`](NVMeshMender.h.md) | Declares the surface implemented above |
| [`mender_input_output.h`](mender_input_output.h.md) | Marshals any engine mesh into the generator's form and reassembles the result, carrying per-vertex data through the splits |
| [`remove_isolated_verts.h`](remove_isolated_verts.h.md) | Rebuilds a mesh with only the referenced vertices, in triangle-visit order |
| [`convert.h`](convert.h.md) | Copies a vector between the engine's type and the generator's |

The directory also carries the vendor's original revision notes, which record that the
new-to-old mapping and the enumerated mode arguments were added after the first release —
the two pieces of the interface that are most obviously right, arrived at last.

## What a rebuild should do differently

- Merge smoothing groups when a triangle can smooth with both of its neighbours. The vendor
  code records it in both and names only one, so it contributes to two averages and
  receives one, which makes the result mildly order-dependent where three or more faces
  meet smoothly.
- Make vector normalization safe against a zero vector, instead of leaving every caller to
  check.
- Derive the degeneracy threshold in the texture-gradient solve from the mesh's scale; it
  is currently an absolute constant.
- Handle non-manifold positions, or reject them: only two edge-neighbours per triangle are
  ever considered.
