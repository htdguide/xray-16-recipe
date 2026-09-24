# src/xrGame/space_restriction_shape.cpp

> Turns one restrictor entity's authored spheres and boxes into a border: the set of navigation vertices that straddle the volume's edge, found by scanning the mesh under each primitive's footprint.

**Needs** — [`space_restriction_shape.h`](space_restriction_shape.h.md) · [`space_restriction_shape_inline.h`](space_restriction_shape_inline.h.md) · [`space_restrictor.h`](space_restrictor.h.md) · [`space_restriction_base.h`](space_restriction_base.h.md) · [`ai_space.h`](ai_space.h.md) · [`xrAICore/Navigation/level_graph.h`](../xrAICore/Navigation/level_graph.h.md) · [`xrAICore/Navigation/graph_engine.h`](../xrAICore/Navigation/graph_engine.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: a bounded scan over the navigation mesh, once per restrictor at spawn

## Purpose

This is where geometry becomes navigation. A restrictor is authored as a handful of spheres
and oriented boxes; the pathfinder searches a grid. The reduction happens exactly once, when
the restrictor spawns, and its result — a list of vertices on the volume's rim — is what
everything else in the family combines, caches and stamps.

## State

```text
RECORD RestrictionShape
  restrictor    : reference to the restrictor entity whose collision volume this is
  is_default    : bool            # part of the level's automatic restriction lists
  border        : list<vertex>    # inherited; the rim of the union of the restrictor's primitives
  initialized   : bool            # inherited; true from construction
```

**Invariants** — a shape is initialized from the moment it is constructed, unlike every
other restriction in the family. Its geometry is already known: the restrictor's collision
form is built during spawn, before registration. That is why registration is the last thing
a restrictor does.

The border is never empty. A restrictor whose volume touches no navigation vertex is an
authoring error — a zone floating above the walkable surface, or one placed off the mesh —
and is reported by name at load time.

## `build_border`

**Contract** — construct the border from every primitive of the restrictor's collision form.
Runs once, from the constructor. Reads the navigation mesh over a bounded region per
primitive. Hard-fails if the result is empty.

```text
FUNCTION build_border()
  border = empty
  FOR EACH primitive IN restrictor.collision_form
    scan_footprint(primitive)

  # A vertex collected on one primitive's rim may be buried inside another.
  # The rim of the union is not the union of the rims.
  REMOVE v FROM border WHERE inside(v, partially = false)

  canonicalize border order        # dedup, then sort by packed horizontal position
  REQUIRE border IS NOT empty

FUNCTION scan_footprint(primitive)
  range = axis-aligned world range of primitive        # see below
  FOR EACH vertex v IN the navigation mesh within range
    # On the rim means: the volume covers part of this cell but not all of it.
    IF inside(v, partially = true) AND NOT inside(v, partially = false)
      APPEND v TO border
```

**Invariants**

- The rim condition is "partially inside AND not fully inside", and both halves are needed.
  Dropping the second half would make the whole interior a border and wall the region off
  from itself; dropping the first would produce nothing.
- The containment tests inside `scan_footprint` are against the **whole restrictor**, not
  against the primitive being scanned. The primitive only supplies the scan bounds. This is
  what makes overlapping primitives compose into one volume instead of several with seams
  where they meet.
- The removal pass runs after every primitive, not per primitive, for the same reason: a
  vertex on primitive A's rim can only be recognized as interior once B has been considered.

**Notes** — the scan bounds are the one place the primitive's own shape is used, and the two
cases differ in a way worth noticing:

- **Sphere** — the range is the centre plus and minus the radius in the horizontal axes
  only; the vertical component is zero. The navigation mesh is indexed horizontally, so a
  vertical extent would not narrow the scan, and including it would only risk excluding
  vertices on a slope beneath the sphere.
- **Box** — the eight corners of the unit cube are transformed by the restrictor's transform
  composed with the box's own, and the range is their axis-aligned extent. An oriented box
  standing diagonally therefore scans a larger region than it occupies, which costs time and
  never costs correctness, because the per-vertex test is exact.

## `inside`, `name`, `sphere`

**Contract** — containment delegates to the restrictor entity's own exact volume test, which
caches its world-space primitives and rejects on a bounding sphere first (see
[`space_restrictor.cpp`](space_restrictor.cpp.md)). The name is the restrictor entity's
name, which is also the key it is filed under. The bounding sphere is the restrictor's
collision bounds in world space, and it is what a composition unions to build its own
rejection sphere.

**Invariants** — the shape holds a reference to a live entity and its answers change when
that entity moves. In practice restrictors do not move, but the entity invalidates its
cached primitives on any transform change, so the containment answers stay right. The
*border* does not: it is built once and never rebuilt. A restrictor that moved after spawn
would keep a border in its old place. Nothing in the shipped data moves one.

## `test_correctness` (checked builds)

**Contract** — verify the border has no gap: flood the navigation mesh from a vertex known
to be inside the volume, with the border stamped as a barrier, and require that the flood
reaches exactly the set of vertices found to be fully inside. Reports failure by restrictor
name. Debug-only.

```text
FUNCTION test_correctness()
  interior = vertices found fully inside during the scan, deduplicated
  IF interior is empty  RETURN         # a rim with no interior is thin, not broken
  stamp border as a mask
  flooded = flood-fill from one interior vertex
  clear the mask
  correct = (count(flooded) == count(interior))
```

**Invariants** — equality, not inequality. A flood that escapes reaches more than the
interior, which means a hole in the rim; a flood that falls short means the interior is not
connected, which means a creature placed in one lobe can never reach the other. Both are
authoring faults that produce creatures frozen in place at run time, and this is the only
automatic check for them. A rebuild should keep it.

**Notes** — the interior list is collected during the same mesh scan that builds the border,
under a second predicate, so the check costs one extra pass over an already-bounded region
rather than a second scan.
