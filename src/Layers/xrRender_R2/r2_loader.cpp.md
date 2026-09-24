# src/Layers/xrRender_R2/r2_loader.cpp

> Brings a level's renderable half into memory — materials, geometry buffers, visuals,
> sector/portal topology, lights — and takes it out again.

**Needs** — [`r2.h`](r2.h.md) ·
[`xrRender/ResourceManager.h`](../xrRender/ResourceManager.h.md) ·
[`xrRender/FBasicVisual.h`](../xrRender/FBasicVisual.h.md) ·
[`xrRender/ModelPool.h`](../xrRender/ModelPool.h.md) ·
[`xrRender/r__sector.h`](../xrRender/r__sector.h.md) ·
[`xrRender/HOM.h`](../xrRender/HOM.h.md) ·
[`xrRender/Light_DB.h`](../xrRender/Light_DB.h.md) ·
[`Common/LevelStructure.hpp`](../../Common/LevelStructure.hpp.md) ·
[`xrCore/FMesh.hpp`](../../xrCore/FMesh.hpp.md) ·
[Seam: Static collision database](../../../SYSTEM-REQUIREMENTS.md#seam-static-collision-database)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: it uploads vertex and index buffers by mapping device memory and
reading raw strides from a stream, and it reads on-disk records as memory images.

## Purpose

The level file is a tree of chunks; this reads the ones the renderer owns. The other half
of the level — collision, navigation, spawns — belongs to chapters 7 and 23. Load order
here is not arbitrary: materials must exist before visuals, visuals before sectors
(because a sector names its root visual), and everything before the lights, which are
placed against sectors.

Unload must be exact. A dedicated server loads none of this, which is why every geometry
step is inside one guard.

## State

Everything it fills lives on the renderer (see [`r2.h`](r2.h.md)); this file owns no state
of its own except the loaded flag.

```text
# The chunks this file reads, in the order it reads them.
RECORD LevelChunks
  shaders   : list<text>    # "<material>/<texture list>" pairs, indexed by position
  visuals   : tree<chunk>   # one sub-chunk per visual, each a model-format record
  sectors   : tree<chunk>   # per sector: its portal indices and its root visual index
  portals   : list<PortalRecord>   # fixed-size records, read as a memory image
  lights, hemisphere        # the static light set
# and, from a sibling file:
  vertex_buffers, index_buffers, slide_windows   # level.geom / level.geomX
```

**Invariants** — material index 0 is reserved and skipped. A material name is `<material
file>/<texture list>` split on the separator; an empty name means the slot is unused. The
portal chunk's byte length must be an exact multiple of the portal record size and the
per-sector portal list an exact multiple of the index size — both are asserted, because a
mismatch means the level was built by an incompatible tool and silently reading garbage
would produce a level that renders almost right.

## `level_load`

**Contract** — loads one level's renderable data. Requires that no level is loaded.
Blocks; reports progress to the loading screen by named stage. Leaves the renderer
marked as loaded.

```text
FUNCTION level_load(level_file)
  begin a deferred-load section          # texture loads are batched until the end
  # --- materials ---
  read the materials chunk; FAIL WITH "level not built correctly" if absent
  FOR EACH entry: split "<material>/<textures>", create the material, store by index

  create the wallmark engine and the detail manager

  IF NOT a dedicated server
      # --- geometry ---
      open level.geom as a stream; load its vertex and index buffers and slide windows
      IF level.geomX exists, load it as the alternate ("fast") geometry set
      # --- visuals ---
      FOR EACH sub-chunk of the visuals chunk
          read its header, create a visual of the declared type, load it, append
      load the detail layer
  # --- topology ---
  load sectors and portals
  load the volumetric-fog volumes, if any and if enabled
  load the software occluder set
  load the lights and the hemisphere lighting
  end the deferred-load section; mark loaded
```

**Notes** — the alternate geometry set is a second copy of the level's vertex and index
buffers with a layout better suited to shadow-only passes; levels that shipped without it
simply do not have the file, and a flag records which state the renderer is in. Every
draw that wants it must ask first.

## `level_unload`

**Contract** — releases everything `level_load` took, in an order that respects the
dependencies: occluders and details first, then topology, then contexts, then lights,
visuals, slide windows, buffers, and finally the materials. Idempotent — returns
immediately if nothing is loaded. Optionally also clears the shared model pool, which is
*not* level-scoped and is kept across level changes unless a setting says otherwise.

**Invariants** — command contexts must be reset before the sector and portal data they
reference is destroyed. The camera's remembered sector must be invalidated, or the next
level's first frame walks from a sector index that means something else.

## `load_geometry_buffers`

**Contract** — reads a vertex-declaration/buffer pair set and an index-buffer set from a
stream, creating device buffers and filling them by mapping. Requires the stream; fails
loudly if the chunk is missing, because a level without geometry is a corrupt level.

```text
FUNCTION load_geometry_buffers(stream, alternate)
  evict cached resources first            # make room; this is the largest allocation
  FOR EACH vertex buffer in the chunk
      peek the declaration, measure its length, then read it properly
      read the vertex count; derive the vertex size from the declaration
      create a device buffer of count*size, map it, read straight into it, unmap
  FOR EACH index buffer in the chunk
      read the index count; create a buffer of count*2, map, read, unmap
```

**Notes** — the declaration is read twice: once to measure (its length is implied by a
terminator, not stored) and once for real. The vertex *size* is derived from the
declaration rather than stored, so the declaration is the authority on layout — which is
the frozen part. Indices are unconditionally sixteen bits wide, which caps a single
buffer at 65536 vertices and is why levels split their geometry across many buffers.

Reading directly into mapped device memory rather than into a staging array is the point
of streaming the file: a level's geometry is the single largest allocation the engine
makes, and doubling it during load is what a rebuild must avoid.

## `load_visuals`

**Contract** — walks the visuals chunk's numbered sub-chunks in order, reads each one's
model-format header, asks the model pool for an instance of the declared type, and lets it
load itself. The resulting list is indexed by position, and those indices appear in the
sector records.

## `load_sectors`

**Contract** — reads the sector/portal topology, builds or loads a cached collision model
over the portal polygons, and installs the result into every command context. Requires the
visuals to be loaded already.

```text
FUNCTION load_sectors(level_file)
  portal_count = portal chunk length / portal record size     # assert exact
  FOR EACH sector sub-chunk
      read its portal index list
      read its root visual index
      volume = the root visual's bounding box volume
      IF volume > best so far, remember this sector as the largest
  # --- the portal collision model ---
  IF portal_count > 0
      identify the model by a checksum over the portal chunk
      IF a cache file exists and its checksum matches, deserialize it
      ELSE
          triangulate every portal polygon as a fan, tagged with its portal index
          IF fewer than two triangles resulted, insert one degenerate triangle far away
          build the tree; write the cache
  FOR EACH command context: reset it and install the sectors and portals
  invalidate the remembered camera sector
```

**Invariants** — the collision model must contain at least two triangles; the tree builder
does not accept fewer. A level whose portals all degenerate gets one synthetic triangle
placed twenty kilometres from the origin, far outside any playable space, purely to
satisfy that.

**Notes** — the "largest sector" is a stand-in for the outdoor sector. Shadow passes for
the sun and the rain start their walk from it, because a directional light's virtual
position is outside the level and has no sector of its own. The comment in the source
calls it a hack and it is one: the level format does not mark which sector is outdoors, so
the biggest bounding box is used as a proxy. It is right for every shipped level and would
be wrong for a level with one enormous indoor volume. A rebuild should record the outdoor
sector explicitly if it controls the level format.

The portal-model cache is keyed by a checksum of the source chunk and can be bypassed or
have its checksum check skipped by command-line switches, which exist because building the
tree is the slowest part of loading a large level.

## `load_slide_windows`

**Contract** — reads the progressive-mesh index ranges: for each entry, four reserved
words, a count, and that many fixed-size records. Optional; absent on levels without
progressive geometry. Frees any previously held set first.

## `load_lights`

**Contract** — hands the light chunk and the hemisphere-lighting chunk to the light
database. One line; recorded because it fixes the load *order* — lights after sectors.

## `load_fog_volumes`

**Contract** — reads the level's volumetric-fog volumes from a sidecar file, if the file
exists and volumetric fog is enabled, and attaches each one as a child of the root visual
of the sector containing its centre. Version-gated: only one version is accepted and
anything else is ignored rather than guessed at.

**Notes** — attaching the volume to a sector's visual hierarchy rather than keeping a
separate list is what makes fog volumes participate in the portal walk for free. It also
means the sector's root visual must be a hierarchy node, which is asserted.
