# src/xrCDB/xr_area.cpp

> Loads a level's collision world — geometry from the level file, tree from the
> level file, a cache file, or a fresh build — and owns it for the level's lifetime.

**Needs** — [`xr_area.h`](xr_area.h.md) · [`xrCDB.h`](xrCDB.h.md) · [`ISpatial.h`](ISpatial.h.md) · [`Common/LevelStructure.hpp`](../Common/LevelStructure.hpp.md) · [Seam: Static collision database](../../SYSTEM-REQUIREMENTS.md#seam-static-collision-database)
**Used by** — reached through its declarations in [`xr_area.h`](xr_area.h.md); callers name that, not this file.
**Tier floor** — T1: the collision file is read as a memory image — the header is taken
whole and the two arrays are addressed by pointing at the mapped region, not parsed field by
field.

## Purpose

The object space is where the collision module meets the level lifecycle. This file answers
one question — *where does the tree for this level come from* — and there are three
answers, tried in order. It also declares the per-thread query scratch that every query in
the module uses.

## State

```text
RECORD ObjectSpace
  spatial_index   : reference to the moving-object octree     # borrowed, not owned
  static_model    : Model                                     # owned, by value
  bounding_volume : box                                       # the level's extent

# per thread, not per object space
THREAD-LOCAL query_handle   : QueryHandle       # named "object space"
THREAD-LOCAL scratch_hits   : HitSet
THREAD-LOCAL scratch_objs   : list<Spatial>
```

**The three thread-local buffers are the concurrency contract.** The static model is
immutable once loaded and the octree takes its own lock, so the only per-query mutable state
is these three, and giving each thread its own removes the last reason to serialize queries.
A rebuild in a tier with cheap thread-locals keeps this; one without should pass a query
context down instead, which is the same decision made explicit.

## the level collision file

Frozen format. The file is a header followed by two arrays, and — in levels built since the
cache was introduced — a trailing block holding a prebuilt tree.

```text
RECORD CollisionFileHeader
  version    : int (32-bit)     # must equal 4; anything else is refused, not converted
  vert_count : int (32-bit)
  face_count : int (32-bit)
  bounds     : box              # min and max, six floats
# then: vert_count vectors, then face_count triangles (16 bytes each)
# then, optionally: a cache-format version int, and if it is the current one,
#                   the game's material table and the serialized tree
```

Version 4 of this file is asserted, not negotiated: an older level is a different world and
guessing at it would be worse than refusing.

## `ObjectSpace.load`

**Contract** — opens the level's collision file (by default the one named `level.cform`
under the level's root), reads it, and leaves the object space ready to query. Fails hard if
the file is absent or its version is wrong — a level without collision is not a level.

```text
FUNCTION load(reader, hooks) -> ()
  IF caching enabled: static_model.source_crc <- checksum over the whole file
  header <- read CollisionFileHeader
  verts  <- the region right after the header          # borrowed, not copied
  tris   <- the region right after the vertices        # borrowed, not copied
  advance past both arrays
  embedded <- IF bytes remain AND the next int is the current cache version
                THEN the reader positioned there ELSE none
  create(verts, tris, header, hooks, embedded)
```

**The checksum is taken over the whole file, before anything is parsed.** It is the identity
of *this geometry*, and it is what a cache file written on a previous run is validated
against. Taking it over the file rather than over the parsed arrays means a change anywhere
in the file — including in the embedded tree — invalidates the cache, which is the
conservative direction.

## `ObjectSpace.create`

**Contract** — installs the geometry and gets a tree, by the first of three routes that
works. Takes the four hooks, so the game layer can inject its material remapping and its own
cache-validity data without this module knowing what a material is.

```text
FUNCTION create(verts, tris, header, hooks, embedded) -> ()
  REQUIRE header.version == 4
  cache_path <- under the user's data root, keyed by the level's name

  IF caching enabled AND embedded is present
     # route 1 -- the tree shipped inside the level file
     static_model.load_geom(verts, tris)          # geometry only; no build
     table <- read (material id -> material name) pairs from embedded
     hooks.remap_materials(static_model.tris, table)
     static_model.load_tree(embedded)

  ELSE IF caching enabled AND cache_path exists AND static_model.deserialize(cache_path, hooks.check)
     # route 2 -- a tree this machine built on an earlier run
     (nothing more to do; deserialize installed geometry and tree together)

  ELSE
     # route 3 -- build it
     static_model.build(verts, tris, hooks.fixup)
     IF caching enabled: static_model.serialize(cache_path, hooks.extra)

  bounding_volume <- header.bounds
```

**Route 1 needs the remapping hook and the other two do not**, and that asymmetry is the
interesting part. A tree is built over triangles whose material ids have already been
rewritten into the running material library's numbering, so routes 2 and 3 get correct ids
for free — route 3 does the rewrite itself (the build fix-up hook), and route 2's cache
holds already-rewritten triangles and is invalidated by the material library's own checksum,
which the deserialize check verifies. Route 1's triangles come from the level file with the
*level's* numbering, and the level does not know what the current material library looks
like. So the embedded block carries a table of id-to-*name* pairs, and the remapping hook
resolves names against the live library. Names, not ids, because a name survives an edit to
the material file and an id does not.

**The cache is written under the user's data root, not next to the level.** The game
installation is read-only in the general case, and the cache is derived data. Keying it by
the level's name means one cache per level, overwritten whenever it goes stale.

Two switches disable pieces of this: one turns caching off entirely (always route 3, never
write), and one skips the cache's checksum verification. Both are developer affordances for
people regenerating geometry, and a rebuild may drop both — but not the checksums
themselves, which are the only thing standing between a stale cache file and a wild pointer
chase inside every query.

## `ObjectSpace.nearest`

**Contract** — collects the game objects whose bounding sphere meets a sphere around a
point, excluding one named object. Asks the moving-object octree for a *box* of the right
size, then filters the candidates by the actual sphere. Returns how many were found.

The box-then-sphere order is the usual one: the octree's box walk is cheap and conservative,
and the exact sphere test runs only on what it admits. Three entry points differ only in
whose candidate buffer they borrow — the caller's, or the thread's scratch.

## `ObjectSpace.dump_statistics`

**Contract** — forwards to the thread's query handle, so the overlay shows *this* thread's
query cost. Instrumentation.

## Notes

The static model and the moving-object index have different owners: the model is a member
here, the octree is borrowed. That is right — the octree outlives any one level, since
objects exist before a level's geometry is loaded and after it is dropped — but it means the
object space cannot be treated as owning "the collision world" as a whole. A rebuild should
keep the two lifetimes distinct for the same reason.
