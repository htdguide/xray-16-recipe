# src/xrGame/obstacles_query.cpp

> Maintains one creature's blocked-vertex set: the union of the navigation vertices its known obstacles cover, kept fresh with a checksum rather than a rebuild.

**Needs** — [`obstacles_query.h`](obstacles_query.h.md) · [`obstacles_query_inline.h`](obstacles_query_inline.h.md) · [`ai_obstacle.h`](ai_obstacle.h.md) · [`GameObject.h`](GameObject.h.md) · [`ai_space.h`](ai_space.h.md) · [`xrAICore/Navigation/level_graph.h`](../xrAICore/Navigation/level_graph.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: set algebra over sorted vertex lists, per creature per path

## Purpose

An obstacle is an object plus the set of navigation vertices it makes unwalkable. A creature
usually has several, and its pathfinder needs their *union*. Recomputing that union whenever
anything moves would be prohibitive, so this file builds it lazily and guards it with a
checksum: an obstacle that moved but is nowhere near the creature does not trigger a rebuild.

The checksum is the design. Everything else on this page is set algebra in service of it.

## State

See [`obstacles_query.h`](obstacles_query.h.md). The per-object stored value is that object's
obstacle checksum *as of the last recomputation*, which is what makes change detection a
comparison rather than a recomputation.

## `compute_area`

**Contract** — rebuilds the cached union and the combined checksum from scratch. Marks the
query fresh. Called only through the freshness protocol, never directly by a consumer.

```text
FUNCTION compute_area()
  actual := true
  area := empty
  checksum := 0
  FOR EACH (object, stored_checksum) IN obstacles
    area := set_union(area, object's obstacle area)
    stored_checksum := object's current obstacle checksum
    checksum := checksum XOR stored_checksum
```

**Invariants** — the combined checksum is the **exclusive-or** of the per-object checksums, so
it is order-independent and incrementally meaningful, but it is also *cancelling*: two objects
with equal checksums contribute nothing. Two different obstacle sets can therefore collide,
which is why the equality comparison checks the area as well and does not trust the checksum
alone.

The union is a sorted-list merge, so every obstacle's vertex area must be sorted. That is an
obligation on the obstacle producer, not enforced here.

**Notes** — the merge allocates a destination of the combined size and shrinks it to the
union's actual size, so each merge is one allocation and one linear pass. With *n* obstacles
this is *n* merges over a growing result; a rebuild merging all *n* at once would be faster
but the counts are small.

## `objects_changed`

**Contract** — answers whether any known obstacle has both changed since the last
recomputation **and** is within a given radius of a given point. Reads only.

```text
FUNCTION objects_changed(position, radius) -> bool
  FOR EACH (object, stored_checksum) IN obstacles
    IF object's current checksum == stored_checksum THEN CONTINUE   # unchanged
    IF object's obstacle is farther than radius from position THEN CONTINUE
    RETURN true
  RETURN false
```

**Invariants** — both conditions are required, and the *distance* one is what makes the whole
scheme pay. A door opening across the level changes its obstacle's checksum every frame; it
must not invalidate the path of a creature two rooms away. The position and radius the caller
supplies are its own position and its interest radius, so "relevant" means "near me".

## `remove_objects`

**Contract** — first brings the query up to date for the given sphere, then drops every
obstacle whose *entire* vertex area lies outside that sphere, then recomputes. Answers whether
the checksum changed.

```text
FUNCTION remove_objects(position, radius) -> bool
  update_objects(position, radius)
  drop every obstacle for which no vertex of its area is within radius of position
  IF nothing was dropped THEN RETURN false
  actual := false
  before := checksum
  compute_area()
  RETURN checksum != before
```

**Invariants** — the too-far test is over the obstacle's **vertices**, not its object position,
and it keeps the obstacle if *any* one vertex is inside. A long obstacle — a fence, a parked
truck — whose origin is far away but whose blocked span reaches the creature must be kept. A
rebuild testing the object's own position instead will let creatures path through the far end
of large obstacles.

The test compares squared distances against a squared radius, avoiding a square root per
vertex; with a few obstacles each covering tens of vertices this is the inner loop of the
whole file.

## `merge`

**Contract** — three forms. The object form unions another query's objects into this one, one
offer at a time, so the freshness protocol's duplicate check applies. The area form is the
sorted-list union used by the recomputation. The proximity form unions the objects and then
decides, from the freshness state, whether anything actually changed.

```text
FUNCTION merge(position, radius, other) -> bool     # "did the result change"
  merge(other's objects)
  IF still fresh                                     # nothing new was added
    IF NOT objects_changed(position, radius) THEN RETURN false
    update_objects(position, radius)
    RETURN true
  before := checksum
  compute_area()
  RETURN checksum != before
```

**Invariants** — "still fresh after the merge" means every offered object was already present,
which is the common case when two creatures keep exchanging the same obstacle set. In that
case the expensive recomputation is skipped entirely and only the cheap proximity check runs.

**Notes** — the proximity form returns true after a successful proximity update without
re-checking the checksum, while the other branch does check. The two branches therefore mean
slightly different things by "changed"; the difference is a false positive in the first
branch, which costs a replan and nothing worse.

## `set_intersection`

**Contract** — keeps only the obstacles also present in another query. Clears the freshness
flag if anything was dropped; leaves it alone if nothing was.

**Invariants** — the conditional invalidation is the same economy as everywhere else on this
page: an intersection that removes nothing must not force a recomputation.

**Notes** — the intersection is computed into stack scratch and written back over the set,
which requires both sets to be sorted by the same key — object address. Sorting by address
means the set's iteration order is allocation-dependent and not reproducible between runs.
That is invisible here because the operations are order-independent (the checksum is an
exclusive-or, the area a set union), but a rebuild introducing any order-dependent step over
this set inherits a determinism hazard.

## `remove_links`

**Contract** — drops one named object from the set, if present. Does **not** clear the
freshness flag or recompute.

**Invariants** — this is the destruction path: it is called when an object is being destroyed
and every reference to it must be dropped before its memory is released. Leaving the cached
area stale is deliberate — the next legitimate read recomputes, and doing a full recomputation
inside a destruction sweep across many creatures would be quadratic.

The consequence is that the cached area may briefly contain vertices blocked by an object that
no longer exists. That is safe: it makes a creature avoid a patch of floor for one path, and
it is the same over-conservatism the static obstacle query already accepts.
