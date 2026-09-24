# src/Layers/xrRender/FSkinned.cpp

> Repack a model file's skinning data into the device layout its bone count calls for, remember which faces each bone owns, and answer the three questions that need a creature's triangles where the animation just put them.

**Needs** — [`FSkinned.h`](FSkinned.h.md) · [`FSkinnedTypes.h`](FSkinnedTypes.h.md) · [`SkeletonX.h`](SkeletonX.h.md) · [`SkeletonCustom.h`](SkeletonCustom.h.md) · [`BufferUtils.h`](BufferUtils.h.md) · [`xrCDB/xrCDB.h`](../../xrCDB/xrCDB.h.md) · [`ResourceManager.h`](ResourceManager.h.md)
**Used by** — reached through its declarations in [`FSkinned.h`](FSkinned.h.md); callers name that, not this file.
**Tier floor** — T1: it maps device buffers for read and write and reinterprets their contents as one of eight byte layouts.

## Purpose

Skinned models are where the renderer's data and the simulation's data are the same data — the chapter-4 README's phrase. This file is the whole of that overlap on the renderer's side: it builds the skinned vertex buffer, and it provides the three read-back paths that let physics, damage and decals reach a creature's *animated* surface.

## The render mode — one decision made per model, at load

```text
The skeleton's own load decides a RENDER MODE for the whole model, from the
maximum number of bones influencing any one of its vertices and from what
the device and the settings allow:

  software skinning                    # the processor skins into a dynamic buffer
  one / two / three / four bones       # quantized device layouts
  the same four, full precision        # float positions and texture coordinates
  "single"                             # one bone, treated as one-bone skinning

Every path in this file switches on that mode and nothing else.
```

**Invariants**

- The mode is a property of the **model**, fixed at load, never per frame and never per vertex. A model with one four-bone vertex is a four-bone model throughout, and every vertex in it pays for four indices. That is the price of one vertex layout per buffer.
- Software skinning draws from the renderer's **shared dynamic vertex stream**, not a buffer of its own — the skinned result is produced fresh each frame and thrown away. Every other mode owns a static buffer built once at load, and the skinning happens in the vertex program.
- The "single" mode is one-bone skinning by another name. It exists so the material system can select a cheaper shader variant for a rigid attachment, and it takes the one-bone layout unchanged.

## `load(name, stream, flags)`

**Contract** — reads the skinning data, then loads as an ordinary model **with vertices suppressed**, then builds the device vertex buffer from the skinning data in the mode's layout.

```text
FUNCTION load(name, stream, flags)
  read the skinning header (which sets the render mode) and the vertex count
  remember where the raw skinned vertices begin in the stream
  load as an ordinary model, suppressing its vertex handling
  tell the material system which skinning variant this model needs
  vertex base = 0                      # a skinned model never shares a buffer
  build the device vertices from the raw ones, in the mode's layout
```

**Invariants**

- **The ordinary load is told not to read vertices**, because the model file's vertex block is in the *skinning* format, not a device format. The base class would read it as opaque bytes and build the wrong buffer. This is the reason the suppression flag exists at all.
- The repacking loop reads one record and writes one record, converting quantization and moving bone indices into their packed positions. It is straight-line, one pass, and it allocates only the destination buffer.
- **The skinned vertex buffer is created readable.** That is a per-model system-memory cost paid by every creature in the game, and it buys the three read-back paths below. It is the single largest memory consequence of a decision in this file, and a rebuild that drops decals-on-creatures and bone-accurate picking can drop it.
- A parallel version of the repacking loop exists behind a build switch, with a note from its author saying it is always slower on the shipped models and has not been tested. The threshold it would engage at is ten thousand vertices, which no shipped model reaches. It is dead.

## `after_load(skeleton, child_index)` and the bone face lists

**Contract** — records which of this mesh's triangles each bone influences, by walking every index, reading each vertex's bone numbers, and appending the triangle to each of those bones' lists. Runs once per model at load. Maps the vertex and index buffers for reading.

```text
FUNCTION collect_bone_faces(mesh, index_base, index_count)
  FOR EACH index position idx IN the range
    vertex = vertices[indices[idx]]
    FOR EACH bone influencing that vertex
      append triangle (idx / 3) to that bone's face list for this child
```

**Invariants**

- The face lists are stored **on the bone, keyed by which child mesh they came from**. A creature is several child meshes and a bone spans them; the key is what keeps them apart.
- **A triangle is appended once per vertex per bone, not once per triangle.** Three vertices of a triangle sharing a bone append it three times. The lists therefore contain duplicates, and the consumers below tolerate them — a decal cut three times against the same triangle produces the same decal. This is a real inefficiency in a load-time structure that is then walked at run time, and a rebuild should deduplicate.
- The simplifying variant collects faces from **the most detailed slide window only**. Decals and picking always work against full detail, whatever is being drawn. That is correct — a bullet hole's position must not depend on how far away the creature was — and it means the lower windows' triangles are never in any bone's list.

## `pick_bone(result, distance, origin, direction, bone)`

**Contract** — intersects a ray against one bone's faces, with each vertex transformed by its bones' *current* render transforms. Returns whether anything was hit and fills in the nearest hit. Maps both buffers for reading; does not allocate.

**Invariants** — The faces tested are only those the named bone influences. The caller has already narrowed to a bone by testing the skeleton's bone bounding volumes, so this is the fine pass of a two-level query. Without the face lists it would have to test the whole mesh per bone.

## `fill_vertices(view, wallmark, normal, size, bone)` — cutting a decal into skin

**Contract** — finds the triangles of one bone that lie within a sphere around the decal's contact point and face the decal's normal, and records them in the decal with their bone bindings and their projected texture coordinates. Maps both buffers for reading.

```text
FUNCTION fill_vertices(view, wallmark, normal, size, bone)
  FOR EACH face IN this bone's face list
    build a triangle from the three vertices' SKINNED positions
    IF the triangle's normal faces away from the decal normal    CONTINUE
    IF the triangle does not intersect the sphere (contact point, size)  CONTINUE
    FOR EACH of its three corners
      project the skinned position through the decal's view matrix
      texture coordinate = (1 + x) / 2, (1 - y) / 2
    record the face in the decal, keeping for each corner:
      its position IN MODEL SPACE, its up-to-four bone numbers, its weights
```

**Invariants**

- **The decal stores model-space positions and bone bindings, not world-space positions.** That is what makes a decal on a creature *follow the animation*: every frame it is re-skinned by the same rule as the mesh. A decal that stored world positions would slide off as soon as the creature moved. This is the single most important invariant in the file.
- The facing test uses the triangle's normal in its **currently animated** pose, so a decal is cut into the surface as it was at the moment of impact.
- The texture projection is the standard "render from the impact direction" trick: the decal carries a view matrix built at the impact, and each corner's coordinate is its projected position mapped from the normalized device range into the unit range. The y is flipped, matching the screen-axis convention noted in [`FVF.h`](FVF.h.md).
- The per-layout code that fills a corner's bone bindings **pads unused slots by repeating the last bone and zeroing the unused weights**, so that the decal's own re-skinning code can always read four bones and three weights without branching. That padding is what makes one decal record serve all four skinning widths.

## `enumerate_bone_vertices(callback, bone)`

**Contract** — hands the caller the model-space position of every vertex of every face the named bone influences. Used by the physics layer to fit a collision shape to a bone.

**Notes** — Unlike the other two, this reports **unskinned** positions. The physics layer wants the bone's geometry in the bone's own frame, which the bind pose is.

## What a rebuild should take from this file

The three live decisions are: **one vertex layout per model chosen by maximum bone count**; **per-bone face lists built at load so that every per-bone query is a short list walk**; and **decals stored as bone-bound model-space triangles**. Everything else — the eight-way switch repeated five times, the parallel dead code, the duplicate face appends — is the cost of writing those three decisions without a way to abstract over the layout.
