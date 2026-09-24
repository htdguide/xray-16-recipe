# src/xrCore/FMesh.hpp

> The visual-model container format: its chunk numbering, its mesh kinds, and its vertex-format identifiers.

**Needs** — [`FMesh.cpp`](FMesh.cpp.md) · [`FS.h`](FS.h.md) · [`xrstring.h`](xrstring.h.md) · [`_vector3d.h`](_vector3d.h.md)
**Used by** — [`FBasicVisual.cpp`](../Layers/xrRender/FBasicVisual.cpp.md) · [`FHierrarhyVisual.cpp`](../Layers/xrRender/FHierrarhyVisual.cpp.md) · [`FLOD.cpp`](../Layers/xrRender/FLOD.cpp.md) · [`FProgressive.cpp`](../Layers/xrRender/FProgressive.cpp.md) · [`FProgressive.h`](../Layers/xrRender/FProgressive.h.md) · [`FTreeVisual.cpp`](../Layers/xrRender/FTreeVisual.cpp.md) · [`FTreeVisual.h`](../Layers/xrRender/FTreeVisual.h.md) · [`FVisual.cpp`](../Layers/xrRender/FVisual.cpp.md) · [`ModelPool.cpp`](../Layers/xrRender/ModelPool.cpp.md) · [`SkeletonAnimated.cpp`](../Layers/xrRender/SkeletonAnimated.cpp.md) · [`SkeletonCustom.cpp`](../Layers/xrRender/SkeletonCustom.cpp.md) · [`SkeletonX.cpp`](../Layers/xrRender/SkeletonX.cpp.md) · [`SkeletonMotions.cpp`](Animation/SkeletonMotions.cpp.md) · [`FMesh.cpp`](FMesh.cpp.md) · _and 1 more_
**Tier floor** — T1: it enumerates a frozen on-disk layout, including a vertex-format tag that is compared as an exact 32-bit value.

## Purpose

Every renderable model the engine loads — static props, skinned characters, trees, level-of-detail impostors, particle effects — is one file in this container format. This header is the format's table of contents: what a chunk identifier means, what kinds of mesh exist, how a skinned vertex's weight count is encoded, and which version tags gate which layout. The reading is done by the renderer's model loader; this file is the shared vocabulary that the loader, the exporters and the animation layer all agree on.

It is a constants-and-records header with almost no code, so it carries the substance; [`FMesh.cpp`](FMesh.cpp.md) holds only the one record with a body.

## State — the container

The file is a chunked container in the format of [`FS.cpp`](FS.cpp.md). Chunk identifiers, **frozen**:

| Id | Chunk | Notes |
|---|---|---|
| 1 | header | always present, always first |
| 2 | texture | two zero-terminated strings: texture name, then material-pass name |
| 3 | vertices | vertex-format tag, count, then the vertices |
| 4 | indices | count, then that many 16-bit indices |
| 5 | progressive map | unused in shipped data |
| 6 | sliding-window data | level-of-detail records for progressive meshes |
| 7, 8 | vertex / index container references | unused |
| 9 | children | skinned models only: a run of nested model chunks |
| 10 | child links | a count and that many 32-bit references to external visuals |
| 11 | impostor definition | a flat quad set with five texture channels |
| 12 | tree definition | a transform plus scale and bias vectors for wind animation |
| 13 | bone names | skeleton only |
| 14 | motions | skeleton only: the animation payload |
| 15 | motion parameters | skeleton only: the partition and motion-definition table |
| 16 | inverse-kinematics data | skeleton only: per-bone joint limits and collision shape |
| 17 | user data | skeleton only: free-form configuration text |
| 18 | description | authoring provenance |
| 19 | motion references | skeleton only: names of external animation banks |
| 20 | sliding-window container | shared records for progressive meshes |
| 21 | geometry container | vertex and index buffer together |
| 22 | fast path | a simplified geometry set for shadow and collision passes |
| 23 | level-of-detail description | skeleton only, configuration text |
| 24 | motion references, second form | a counted list rather than a single string |
| 25 | collision vertices | the collision proxy's positions |
| 26 | collision indices | the collision proxy's triangles |

```text
RECORD Header                    # chunk 1, fixed layout
  format_version : int (8-bit)   # currently 4
  type           : int (8-bit)   # which mesh kind, below
  shader_id      : int (16-bit)  # index into the level's material table;
                                 # must not be zero
  bbox_min       : real[3]       # axis-aligned bounds
  bbox_max       : real[3]
  sphere_center  : real[3]       # bounding sphere, for coarse culling
  sphere_radius  : real
```

**Invariants** — the bounding sphere is stored, not derived; it is not necessarily the minimal sphere around the box, and the culling pass relies on the stored value. A shader identifier of zero means the exporter failed to assign a material and the model is unusable.

## `MT` — the mesh kinds

The header's `type` selects which further chunks are meaningful:

```text
ENUM MeshKind
  normal            = 0    # one static mesh
  hierarchy         = 1    # a group of child visuals
  progressive       = 2    # one mesh with a sliding-window level-of-detail chain
  skeleton_animated = 3    # a skinned model that owns motions
  skeleton_geom_pm  = 4    # skinned geometry, progressive
  skeleton_geom_st  = 5    # skinned geometry, static
  lod               = 6    # the far-distance impostor
  tree_static       = 7    # vegetation, wind-animated in the vertex stage
  particle_effect   = 8
  particle_group    = 9
  skeleton_rigid    = 10   # a hierarchy rigidly bound to bones, no skinning
  tree_progressive  = 11
  fluid_volume      = 12
```

## `OGF_SkeletonVertType` — the weight-count tag

The vertices chunk begins with a 32-bit tag naming the vertex layout. For skinned meshes the tag is a small multiple of one fixed magic number, and the multiplier is the number of bone influences per vertex:

```text
one influence    = 1 * 0x12071980
two influences   = 2 * 0x12071980
four influences  = 4 * 0x12071980    # note: 4, not 3
three influences = 3 * 0x12071980    # note: 3 means the NO-weight form
```

**Notes** — the mapping from multiplier to influence count is *not* monotone: multiplier 3 names the un-skinned form and multiplier 5 names the four-influence form. This looks like an accident of the order the layouts were added, and it is frozen. A rebuild must treat the tag as an opaque enumeration and not compute the influence count from it.

The magic number itself is a date written as digits. It carries no other meaning; its only job is to be a value no plausible count or size could collide with.

## `ogf_desc` — the description chunk

**Contract** — authoring provenance: the source asset path, and three (name, timestamp) pairs for who built, created and last modified the model. Purely informational; the engine reads it and shows it in the tools. Serialization is in [`FMesh.cpp`](FMesh.cpp.md).

## `FSlideWindow` / `FSlideWindowItem`

**Contract** — one level of a progressive mesh: a starting index offset, a triangle count and a vertex count. Rendering a given level means drawing `num_tris` triangles starting at `offset`, using the first `num_verts` vertices. An item is an array of these plus its length.

**Invariants** — the windows are ordered coarse to fine or fine to coarse consistently within a file, and consecutive windows share a prefix of the index buffer — that is the whole point: switching level of detail changes a draw range, not a buffer.

**Notes** — the item record carries four reserved words. Nothing reads them. They exist so the in-memory record matched a hardware constant-buffer size on the platform of the day, and a rebuild should drop them.

## Version constants

- **format version 4** — the current model header version.
- **motion-parameters version 4** — the current version of the skeleton's motion-parameter chunk; the loader refuses a higher value and adapts to lower ones (see [`Animation/SkeletonMotions.cpp`](Animation/SkeletonMotions.cpp.md)).

## Could not recover

Chunks 5, 7 and 8 are declared and are marked unused in the source. Whether any shipped archive contains them was not determinable here.
