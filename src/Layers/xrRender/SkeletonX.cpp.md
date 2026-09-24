# src/Layers/xrRender/SkeletonX.cpp

> The skinned mesh: one sub-mesh of a skeletal model, which decides at load time whether the device or the processor will deform it, and then either uploads a bone-matrix array or transforms every vertex itself into the frame's dynamic stream.

**Needs** — [`SkeletonX.h`](SkeletonX.h.md) · [`SkeletonXSkinXW.h`](SkeletonXSkinXW.h.md) · [`SkeletonXVertRender.h`](SkeletonXVertRender.h.md) · [`SkeletonCustom.h`](SkeletonCustom.h.md) · [`R_DStreams.h`](R_DStreams.h.md) · [`R_Backend.h`](R_Backend.h.md) · [`xrCore/FMesh.hpp`](../../xrCore/FMesh.hpp.md) · [`xrCore/Animation/Bone.hpp`](../../xrCore/Animation/Bone.hpp.md) · [`xrCDB/Intersect.hpp`](../../xrCDB/Intersect.hpp.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`SkeletonX.h`](SkeletonX.h.md) · [`SkeletonXVertRender.h`](SkeletonXVertRender.h.md)
**Tier floor** — T1: it points a vertex record directly at a mapped region of the shipped model file, and it writes vertices into device-mapped memory with a fixed byte layout.

## Purpose

A skeletal model is a bone hierarchy ([`SkeletonCustom.cpp`](SkeletonCustom.cpp.md)) plus a set of sub-meshes, each with its own material. This file is one sub-mesh. Its whole job is the *deformation decision*: given how many bones influence each vertex and how many bones the mesh touches in total, can the device do the skinning in the vertex program, or must the processor do it?

That decision is made **once, at load**, not per frame, and it is sticky: it chooses the render mode, it chooses which material variant the compiler will produce, and it chooses whether the mesh keeps a system-memory copy of its vertices at all. Everything else in the file — picking a bone with a ray, laying a decal across a deforming surface — needs those same system-memory vertices, which is why those paths exist only in the modes that keep them.

## State

```text
RECORD SkinnedMesh
  parent      : Skeleton              # the bone hierarchy that owns this sub-mesh; set after load
  child_index : int (16-bit)          # which sub-mesh of the parent this is
  vertices1W, vertices2W, vertices3W, vertices4W : shared array, at most one non-empty
  bones_used  : shared list<int (16-bit)>   # the distinct bone ids this mesh is influenced by
  render_mode : enum (see below)

  # one of these three, depending on the render mode — they share storage
  soft_cache  : (discard_id : int, vertex_count : int, vertex_offset : int)
  single_bone : int                   # the one bone, in the single-bone mode
  bone_span   : int                   # highest bone id plus one, in the hardware modes
```

**Invariants**

- **Exactly one of the four vertex arrays is populated, and only in the software mode.** In every hardware mode all four are empty: the vertices live in an immutable device buffer built by the owning visual, and no system copy is kept. This is why bone picking and decal projection are implemented only against the four software formats — a hardware-skinned mesh has nothing to walk. (The game works around this by picking against the collision proxies instead.)
- The vertex arrays and the used-bone list are **shared by content hash**: two instances of the same model, and two different models that happen to contain byte-identical meshes, hold the same array. The hash is a checksum over the raw bytes. A rebuild must intern on content, not on file name, or a level's memory grows with the number of characters rather than the number of distinct meshes.
- `bone_span` is the **highest bone identifier plus one**, not the number of distinct bones. The constant array the device reads is indexed by bone id directly, so it must span the largest id even when the ids are sparse.
- The three mode-specific fields overlap in storage. Copying a mesh copies all three fields blindly, which is correct only because exactly one is meaningful and the other two are then dead. A rebuild uses a tagged union and loses the hazard.
- The soft cache is only valid while the dynamic vertex ring has not wrapped — see the skinning cache below and [`R_DStreams.cpp`](R_DStreams.cpp.md).

### Render modes

```text
soft                     # the processor skins into the dynamic vertex stream
single                   # one bone for the whole mesh: no skinning at all, just a transform
single_hq
skinning_1b .. 4b        # the device skins, with 1..4 influences per vertex
skinning_1b_hq .. 4b_hq
```

Every hardware mode comes in a plain and a *high quality* form, selected by one global flag. The pair differs only in which material variant is compiled — the mesh data and the matrix upload are identical.

## The shipped vertex formats — frozen

Four vertex layouts, one per influence count, read as a byte image straight out of the model file. All four carry position, normal, tangent and bitangent, and a single texture coordinate pair; they differ in how the bone references and weights are packed, and in **where those fields sit relative to the geometry**.

```text
RECORD vertex, 1 influence          # 60 bytes
  position, normal, tangent, bitangent : 3 reals each
  u, v        : real
  bone        : int (32-bit)        # note the width: this one format uses a full word

RECORD vertex, 2 influences         # 64 bytes
  bone0, bone1 : int (16-bit)       # NOTE: the bone ids come FIRST, before the geometry
  position, normal, tangent, bitangent : 3 reals each
  weight      : real                # weight of bone0; bone1 gets the remainder
  u, v        : real

RECORD vertex, 3 influences         # 70 bytes
  bone[3]     : int (16-bit)
  position, normal, tangent, bitangent : 3 reals each
  weight[2]   : real                # the third weight is implied
  u, v        : real

RECORD vertex, 4 influences         # 76 bytes
  bone[4]     : int (16-bit)
  position, normal, tangent, bitangent : 3 reals each
  weight[3]   : real                # the fourth weight is implied
  u, v        : real
```

**Invariants**

- **An n-influence vertex stores n-1 weights.** The last weight is always `1 - (sum of the stored weights)`. This is not a space optimization that a rebuild may undo silently: it *guarantees* the weights sum to one, so there is no normalization step anywhere and a mesh whose stored weights exceed one produces a negative last weight rather than an error. The shipped data relies on the implicit form.
- The record sizes are not multiples of four (70 and 76 are, 60 and 64 are, but the 3- and 4-influence forms begin with an odd number of 16-bit fields). They are packed to two-byte alignment, and the position field is therefore *not* four-byte aligned in the 3-influence format. The loaders read them unaligned, which the platform assumptions permit.
- Tangent and bitangent are present in every source format and **discarded** by software skinning — see [`SkeletonXVertRender.h`](SkeletonXVertRender.h.md).

## `_Load`

**Contract** — reads the vertex chunk of a sub-mesh, scans it to discover which bones it uses, picks the render mode, and either keeps a shared system copy (software) or leaves the bytes to the caller to upload (hardware). Reports the vertex count. Fatal on an unrecognized vertex format. Allocates the shared arrays; blocks only on the already-mapped file region.

```text
FUNCTION load(name, reader) -> vertex_count
  bone_budget = (device constant registers - 22 - 3) / 3
  IF the forward renderer is configured for software skinning, or the device has no
     programmable path at all
    bone_budget = 0                        # force every mesh to the software path

  find the vertex chunk
  format = read word ; vertex_count = read word
  render_mode = soft
  declare "no skinning" to the material compiler

  scan every vertex of the declared format, collecting
    distinct_bones : the set of bone ids referenced
    highest_bone   : the largest bone id referenced

  IF format is the 1-influence form AND there is exactly one distinct bone
    render_mode = single ; single_bone = that bone
    declare "single bone" to the material compiler
  ELSE IF highest_bone <= bone_budget
    render_mode = the hardware mode for this influence count
    bone_span = highest_bone + 1
    declare "n-bone skinning" to the material compiler
  ELSE
    keep a shared system copy of the vertices, keyed by their checksum
    declare "no skinning" to the material compiler

  IF more than one distinct bone is used
    keep the distinct-bone list, shared by its own checksum
```

**Notes**

- **The bone budget.** Each bone costs three four-component constant registers (a 4×3 affine transform, see below). Twenty-two registers are reserved for everything else a vertex program needs — transforms, lighting terms, fog. A further three are subtracted because some materials in the forward renderer need more headroom than the original reservation allowed; the source records this as a later correction, not an original design. A rebuild computes the same quantity from its own reservation and should treat 22 and 3 as *its* numbers to recompute, not constants to copy.
- The vertex format word accepts **two spellings** for every influence count: an enumerated format identifier from the older model generation, and the bare integers 1 to 4. Both occur in shipped data and both must be accepted.
- The declaration to the material compiler is what selects the skinning variant of the shader source. It is a side effect on a global, made during a load — the mesh and its material are compiled in lockstep, and a rebuild that separates the two must pass the influence count explicitly instead.
- **The single-bone case is a mode, not an optimization of the 1-influence case.** When an entire sub-mesh follows one bone there is nothing to blend: the bone's transform is composed into the world transform and the mesh draws as ordinary static geometry. This is the common case for weapon attachments and detachable parts, and it costs zero constant registers.
- When the device has no programmable path, *everything* falls to software, including the single-bone case — that branch is skipped entirely.

## `_Render` — the hardware paths

**Contract** — issues one indexed triangle-list draw for this sub-mesh from the already-bound geometry. Sets up whatever the render mode requires first. Accumulates per-mode vertex counts into the frame statistics. Does not allocate.

```text
FUNCTION render(backend, geometry, vertex_count, index_offset, primitive_count)
  CASE render_mode OF
    soft:
      render_soft(...)                      # below

    single, single_hq:
      world = current world transform composed with parent.transform_of(single_bone)
      set world transform
      draw

    any n-bone hardware mode:
      array = the named bone-matrix constant array          # named "sbones_array" — FROZEN
      FOR bone_id FROM 0 TO bone_span - 1
        M = parent.transform_of(bone_id)
        write 3 registers at (bone_id * 3) + 0, +1, +2
      draw
```

### The bone-matrix upload — frozen

This is the convention the shipped vertex programs are written against, and it is the one thing in the file a rebuild cannot choose freely:

```text
register (bone_id * 3) + 0  =  ( M.row1.x, M.row2.x, M.row3.x, M.row4.x )
register (bone_id * 3) + 1  =  ( M.row1.y, M.row2.y, M.row3.y, M.row4.y )
register (bone_id * 3) + 2  =  ( M.row1.z, M.row2.z, M.row3.z, M.row4.z )
```

Three registers per bone, holding the **transposed** upper 4×3 of the bone's transform — so each register is a *column* of the original, and a shader skins a vertex with three four-component dot products against the position extended by one. The fourth column is the identity row of an affine transform and is never sent.

**Invariants**

- **The stride of three is the encoding.** A shader indexes `bone * 3`, and the shipped data's bone ids are raw indices into that array. Nothing remaps them. A rebuild that packs bones differently must regenerate every shipped skinned shader.
- The array is written for **every id from zero up to the span**, including ids the mesh does not use. Skipping the unused ones would require a remap table on both sides; filling them is cheaper than maintaining one.
- The upload happens **per draw**, not per frame. A model drawn three times in a frame uploads its bones three times. That is the cost of the constant-array approach and the reason the single-bone mode exists.

## `_Render_soft` — the software path

**Contract** — transforms every vertex into the frame's dynamic vertex ring, once per frame at most, then draws from the region it wrote. Maps and unmaps the ring; must not be called with another map outstanding. Timed into the skinning statistic.

```text
FUNCTION render_soft(backend, geometry, vertex_count, index_offset, primitive_count)
  IF cache.discard_id differs from the ring's current discard counter
     OR cache.vertex_count differs from vertex_count
    (destination, offset) = map the vertex ring for vertex_count vertices at this stride
    cache = (ring's discard counter, vertex_count, offset)
    dispatch on which vertex array is populated:
       1, 2, 3 or 4 influences -> the matching skinning routine,
                                  given the parent's bone instance array
    unmap
  set geometry ; draw starting at cache.offset
```

**Notes**

- **The cache is the whole point.** A character is drawn several times per frame — a depth pre-pass, the shading pass, one or more shadow maps — and the deformed vertices are identical every time. Recording the ring's discard counter alongside the offset answers "is what I wrote still there?" exactly, because the counter moves only when the ring wraps and invalidates every outstanding offset. This is the consumer that [`R_DStreams.cpp`](R_DStreams.cpp.md) describes the counter for.
- The vertex count is compared as well as the counter, because the same mesh can be asked to draw a different span (a level-of-detail switch) within one frame without the ring having wrapped.
- Reaching this path with none of the four arrays populated is a hard error. It means a mesh was put in the software mode without a system copy, which the load path cannot produce.

## `has_visible_bones`

**Contract** — true when at least one bone this mesh is attached to is currently visible. In the single-bone mode that is one query; otherwise it walks the distinct-bone list. Used to skip a sub-mesh entirely — the game hides body parts (a severed limb, a helmet) by hiding their bones, and a sub-mesh all of whose bones are hidden must not be drawn.

## `get_pos_bones` — the deformed position of one vertex

**Contract** — computes the skinned world position of a single vertex, given the parent's bone instances. Four forms, one per influence count. Pure. This is the software skinning transform reduced to position only, and it is what the picking and decal paths use instead of the full routines, because they need a handful of vertices rather than all of them.

```text
1 influence : position transformed by the one bone
2 influences: linear interpolation between the two transformed positions, by the stored weight
3 influences: weighted sum, third weight = 1 - w0 - w1
4 influences: weighted sum, fourth weight = 1 - w0 - w1 - w2
```

**Invariants** — the two-influence form interpolates while the three- and four-influence forms accumulate weighted terms. The results agree, and the difference is a rounding-level artifact of the arithmetic order, not a decision.

## `_PickBoneSoft1W` … `4W`

**Contract** — intersect a ray against the faces of one bone's share of this mesh, in the mesh's current deformed pose. Returns the first face hit within the range, together with its geometric normal and distance; the search is over a caller-supplied face list (the faces that bone owns) and a caller-supplied index buffer. Returns false when nothing is hit. Allocates nothing; reads the parent's bone transforms.

```text
FUNCTION pick(range, origin, direction, indices, faces) -> hit?
  FOR EACH face IN faces
    read its three vertex indices (face number times three)
    deform each of the three vertices into world space
    test ray against triangle, accepting back faces
    IF hit AND distance < range
      normal = the triangle's geometric normal
      RETURN hit                         # first acceptable hit wins, not the nearest
  RETURN miss
```

**Notes** — The search returns the **first** face that qualifies, not the nearest. The caller narrows the candidate set by bone first, so the set is small and usually convex from the ray's direction; the engine accepts the approximation. A rebuild that wants the nearest hit must keep searching, and will then not match the original's choice of hit bone in ambiguous cases.

## `_FillVerticesSoft1W` … `4W` — projecting a decal onto a deforming mesh

**Contract** — given a projection matrix, a decal record, a facing normal, a radius and one bone's face list, appends every face of that bone which faces the decal and overlaps its sphere. Each appended face records its vertices **in bind pose** together with their bone references and weights, so the decal deforms with the mesh afterwards rather than being baked to the pose it was created in. Appends to the decal; allocates in it.

```text
FUNCTION fill(projection, decal, facing_normal, radius, indices, faces)
  FOR EACH face IN faces
    FOR EACH of its three vertices
      record bone ids and weights into the face record, padded (see below)
      record the vertex's BIND-POSE position
      deform it to world space for the tests below
    IF the deformed triangle's normal faces away from the decal normal, SKIP
    IF the deformed triangle does not overlap the sphere at the contact point, SKIP
    FOR EACH of its three vertices
      project the deformed position through the projection matrix
      texture u = (1 + projected.x) / 2
      texture v = (1 - projected.y) / 2          # vertical axis flips
    append the face to the decal
```

**Invariants**

- **The decal's face record always carries four bone ids and three weights**, whatever the source format's influence count. Lower counts pad by *replicating the last bone id* into the remaining slots and setting the unused weights to zero. Replication rather than a sentinel is what lets the decal's own deformation code run one unconditional four-influence path: a replicated bone with zero weight contributes nothing.
- The 1-influence form stores **all three weights as zero**, which under the implied-last-weight rule gives the single bone a weight of one. The 2-influence form stores the source weight in the first slot and zero in the rest, giving bone0 its weight and bone1 the remainder — consistent, because slots 2 and 3 replicate bone1.
- Positions are recorded in bind pose and re-deformed later; the deformed positions computed here are used **only** for the facing and overlap tests and for the texture projection. Storing the deformed ones would freeze the decal to the pose it was stamped in.
- The facing test rejects a face whose deformed normal is not at least infinitesimally aligned with the decal's normal — a strictly-greater-than-zero test, so exactly perpendicular faces are rejected.
- The texture coordinate mapping is the standard clip-space-to-texture-space bias, with the vertical axis inverted because texture space runs downward. **Frozen**: the shipped decal materials sample with these coordinates.

## `_Copy` · `AfterLoad` · `SetParent`

**Contract** — `_Copy` clones a mesh from a base by sharing every array and copying the mode fields, leaving the parent unset; `AfterLoad` installs the parent and this mesh's index within it. The split exists because a clone is made before its parent skeleton exists (see [`ModelPool.cpp`](ModelPool.cpp.md)), so the back-pointer is filled in a second step.

## Could not recover

- Why the 1-influence format spends a full 32-bit word on its bone reference when every other format uses 16 bits. It is read back as a 16-bit value everywhere, so the upper half is always zero; the likeliest explanation is that this format predates the multi-influence ones, but nothing says so.
- An earlier, commented-out picking implementation returned the *nearest* hit rather than the first. Whether the change to first-hit was a deliberate optimization or an accident of rewriting is not recorded.
