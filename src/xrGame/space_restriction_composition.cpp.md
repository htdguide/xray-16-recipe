# src/xrGame/space_restriction_composition.cpp

> Several named restrictors as one volume: the union of their shapes, with a single enclosing sphere for cheap rejection and a border that is the rim of the union rather than the concatenation of the rims.

**Needs** — [`space_restriction_composition.h`](space_restriction_composition.h.md) · [`space_restriction_composition_inline.h`](space_restriction_composition_inline.h.md) · [`space_restriction_holder.h`](space_restriction_holder.h.md) · [`space_restriction_bridge.h`](space_restriction_bridge.h.md) · [`space_restriction_base.h`](space_restriction_base.h.md) · [`ai_space.h`](ai_space.h.md) · [`xrAICore/Navigation/level_graph.h`](../xrAICore/Navigation/level_graph.h.md) · [`xrAICore/Navigation/graph_engine.h`](../xrAICore/Navigation/graph_engine.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: a linear volume test behind a sphere reject, built once from member borders

## Purpose

An entity names several restrictors in one list, and the list has to behave like one volume.
This file is that volume. It is deliberately a **union**: a creature restricted to three
regions may be in any of them.

It is also the placeholder. A list naming a restrictor that has not spawned resolves to a
composition that never finishes initializing, and an uninitialized composition answers
*inside* to everything — which propagates upward as "this restriction is not real yet", and
keeps the entity unrestricted rather than trapped. That double duty is why the type exists
even for a single name.

## State

```text
RECORD RestrictionComposition
  member_names : text                  # normalized, comma-joined; also the identity
  members      : list<restriction>     # resolved handles, one per name
  bounds       : sphere                # encloses every member's own bounding sphere
  border       : list<vertex>          # inherited; the rim of the union
  initialized  : bool                  # inherited
  holder       : reference to the registry that resolves names
```

**Invariants**

- Every member is a **shape**, never another composition. Names in the list are single
  restrictor names, each of which resolves either to a real shape or to a single-name
  placeholder — and a placeholder is not initialized, which makes initialization bail
  before any member is asked for its bounding sphere. Compositions do not report a bounding
  sphere at all; asking for one is a hard failure. A rebuild that allows nested
  compositions must supply that sphere.
- A composition of exactly **one** name never initializes. That is not an oversight: it is
  how "this named restrictor does not exist yet" is represented, and it is the state a
  bridge is swapped back to when a restrictor despawns.
- `initialized` must be set **before** the border's containment filter runs, because the
  filter asks this same object whether vertices are inside it, and an uninitialized
  composition would answer by re-entering initialization.

## `initialize`

**Contract** — resolve nothing (the names were resolved on demand), but require every
member to have built its border, then merge. Either completes and marks itself
initialized, or returns having changed nothing, to be retried on the next query. Allocates
a scratch array of member bounding spheres.

```text
FUNCTION initialize()
  n = number of names
  REQUIRE n > 0
  IF n == 1  RETURN                        # the placeholder case; stays uninitialized

  FOR EACH name
    IF NOT holder.restriction(name).initialized  RETURN   # not ready; retry later

  FOR EACH name
    m = holder.restriction(name)
    members.append(m)
    border.prepend(m.border)               # concatenated; filtered below
    collect m.bounds

  bounds = enclosing_sphere(collected)

  initialized = true                       # BEFORE the filter: the filter re-enters inside()

  # The rim of a union is not the union of the rims: where two members overlap,
  # one member's rim runs through the interior of the other and is not a boundary.
  REMOVE v FROM border WHERE inside(v, partially = false)

  canonicalize border order                # dedup, then sort by packed horizontal position

FUNCTION enclosing_sphere(spheres) -> sphere
  box    = axis-aligned box covering every sphere
  centre = centre of box
  radius = max over spheres OF (distance(centre, s.centre) + s.radius)
  RETURN sphere(centre, radius + epsilon)
```

**Invariants** — the enclosing sphere must genuinely enclose, because it is used as a
rejection test and a sphere that is too small silently excludes real containment. It is
deliberately *not* the minimum enclosing sphere: the centre of the members' bounding box is
a cheap, good-enough centre, and the radius is then chosen to make the result correct
whatever that centre is. Computing a true minimal sphere would be exact and pointless —
restrictor volumes are authored a level apart, not packed.

The epsilon added to the radius covers the case where a member sphere is exactly tangent,
which the containment test would otherwise reject on a rounding boundary.

**Notes** — the filter pass is the only part of this that is real geometry, and it is worth
being clear about what it removes. Two overlapping restrictors each contribute a full rim;
the arcs that lie inside the other restrictor are interior to the union and must go, or the
pathfinder would find a wall running through the middle of a region the entity may freely
cross.

## `inside`

**Contract** — is a sphere within the union? Initializes on demand and answers **true** when
still uninitialized. Rejects on the enclosing sphere before touching members.

```text
FUNCTION inside(sphere) -> bool
  IF NOT initialized
    initialize()
    IF NOT initialized  RETURN true        # unresolved: permissive, see below
  IF NOT bounds.intersects(sphere)  RETURN false
  RETURN ANY m IN members SATISFIES m.inside(sphere)
```

**Invariants** — the permissive answer for an unresolved composition is safe only because
of what sits above it: a composed restriction refuses to mark *itself* initialized while
any member is unresolved, and an uninitialized restriction answers *accessible* to every
query. So the permissiveness never reaches a decision. A rebuild that lets an unresolved
composition's answer reach the accessibility test directly gets it backwards on the
forbidden side, where "inside everything" means "nowhere is legal".

## `shape`, `default_restrictor`, `name`, `sphere`

**Contract** — a composition reports that it is not a shape (which is what exempts shapes
from garbage collection, so compositions *are* collectable) and not a default restrictor.
Its name is its normalized member list, which is also its cache key in the registry.
Asking a composition for its own bounding sphere is a hard failure by design — see the
nesting invariant above.

## `test_correctness` (checked builds)

**Contract** — verify that the merged border actually encloses the union: flood the
navigation mesh outward from a vertex known to be inside each member, with the border
stamped as a barrier, and check that the flood is contained. Reports a per-restrictor
failure by name. Debug-only.

```text
FOR EACH member
  REQUIRE member has recorded interior vertices
  stamp border as a mask
  flooded = flood-fill from one of the member's interior vertices
  clear the mask
  correct = (flooded reached the flood cap) OR (interior vertex count <= flooded count)
```

**Notes** — a flood that reaches the cap is treated as correct rather than as a leak, which
reads backwards until you see that the cap is what a *leak* looks like: an unbounded flood
escapes into the level and stops on the cap. Treating that as success makes the test
useless for the worst failure it could catch, and is the weakest part of the check. The
composition-level test also compares counts with an inequality where the per-shape test in
[`space_restriction_shape.cpp`](space_restriction_shape.cpp.md) compares them with
equality, because a union's flood legitimately covers more vertices than any one member's
interior list. A rebuild should keep the check and fix the cap case.

## Notes

A process-wide counter of live compositions is incremented on construction and decremented
on destruction. Nothing reads it in the shipped build; it is a leak tripwire left in place.

In checked builds the constructor also verifies that a single-name composition actually
names an object that is a restrictor and whose type is a real restrictor type, catching the
common authoring mistake of restricting an entity with the name of some other object.
