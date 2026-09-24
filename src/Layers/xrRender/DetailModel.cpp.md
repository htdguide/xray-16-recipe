# src/Layers/xrRender/DetailModel.cpp

> Loads one grass model and stamps transformed copies of it into a shared draw buffer, two indices at a time.

**Needs** — [`DetailModel.h`](DetailModel.h.md) · [`DetailFormat.h`](DetailFormat.h.md) · [`xrStripify.h`](xrStripify.h.md) · [`Shader.h`](Shader.h.md) · [`HWCaps.h`](HWCaps.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: `transfer` writes vertex and index records into a mapped device buffer and rewrites the index loop to move two 16-bit indices per 32-bit operation.

## Purpose

One entry in a level's detail-object table. Everything about it is small — a few dozen vertices, one material, a scale range — and everything about it is called tens of thousands of times a frame. The file is entirely the load path and the stamping loop.

## `load(reader)`

**Contract** — reads one model record and leaves the object ready to stamp. Allocates the vertex and index arrays; blocks on the material resolution, which may compile a shader. Fails hard on malformed data rather than degrading.

```text
FUNCTION load(reader)
  shader_name  = reader.read_string()
  texture_name = reader.read_string()
  material     = resolve_material(shader_name, texture_name)

  flags      = reader.read_int()      # currently one bit: "this model does not wave"
  min_scale  = reader.read_real()
  max_scale  = reader.read_real()
  vertex_count = reader.read_int()
  index_count  = reader.read_int()
  REQUIRE index_count is divisible by 3

  vertices = reader.read(vertex_count records of {position, u, v})
  indices  = reader.read(index_count 16-bit integers)

  bounding_box = the box containing every vertex position
  bounding_sphere = the sphere containing that box

  optimize()        # see below
```

**Invariants**

- The bounds are *computed*, never read from the file. The file could carry them and does not — the models are tiny and this removes a class of authoring error where the bounds disagree with the geometry. Since the detail layer culls and fades by these bounds, a disagreement would show as grass vanishing early.
- A model's material is named by a *pair*: the material template name and the texture name. This is the shipped material system's universal addressing — see [`ResourceManager.cpp`](ResourceManager.cpp.md) — and is why several grass models sharing a template still get distinct materials.
- Indices are validated against the vertex count only in a checked build. The shipped data is trusted.

## `optimize()`

**Contract** — reorders triangles for the hardware post-transform vertex cache and permutes the vertices to match, but only if the reordering actually reduces the simulated cache miss count. Runs once per model at level load. Allocates temporaries.

```text
FUNCTION optimize()
  cache_size = device capability: post-transform vertex cache entries
  before = simulate_cache_misses(indices, cache_size)
  (new_indices, permutation) = stripify(indices, cache_size)
  after  = simulate_cache_misses(new_indices, cache_size)
  IF after < before
    indices  = new_indices
    vertices = vertices permuted by permutation
```

**Notes** — The guard is the interesting part: the optimizer is allowed to *not help*, and on a thirty-vertex tuft it frequently does not. Applying its output unconditionally would sometimes make things worse. This is the same simulate-then-accept pattern the level geometry uses; see [`xrStripify.cpp`](xrStripify.cpp.md).

The optimization runs only on one backend in the original, gated by a build condition. Nothing about it is backend-specific — the gate is an artefact of which backend the code was added for. A rebuild should run it on all paths or none.

## `transfer(transform, out_vertices, colour, out_indices, index_offset)`

**Contract** — writes one instance's worth of vertices and indices into the caller's mapped buffers. See [`IRenderDetailModel.h`](IRenderDetailModel.h.md) for the interface contract. The variant taking a texture-coordinate offset adds it to every vertex's coordinates.

**Invariants** — The caller's index offset must fit in 16 bits. This is the hard cap on how many vertices may be accumulated in one batch before flushing, and it is asserted rather than handled.

## The index-shifting loop

**Contract** — adds a constant to every index in the model, writing the results into the destination.

```text
FUNCTION shift_indices(destination, offset)
  # The offset is replicated into both halves of a machine word, and the
  # indices are copied a pair at a time: two 16-bit adds become one 32-bit add.
  # There is no carry between the halves because the sum of an index and the
  # offset is bounded by the same 16-bit limit the caller already asserted.
  packed = offset in both halves of a 32-bit word
  FOR EACH aligned pair of indices
    write (pair + packed)
  IF the count is odd
    write the last index + offset
```

**Notes** — This is the one genuinely hand-optimized loop in the detail layer, and the original labels it a dirty hack. The decision it encodes is load-bearing and survives translation: *index rebasing is the inner loop of the grass layer, and it should move two indices per operation.* The reason it is safe — that no carry can cross the halfword boundary because the caller has already bounded the sum — is the invariant worth writing down; everything else about it is a spelling. A rebuild in a language with wide-vector operations should do four or eight at a time and drop the odd-count special case into the same machinery.
