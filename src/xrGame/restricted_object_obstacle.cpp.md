# src/xrGame/restricted_object_obstacle.cpp

> Masks the navigation vertices blocked by other objects out of the graph for the duration of one path — while never masking the path's own endpoints.

**Needs** — [`restricted_object_obstacle.h`](restricted_object_obstacle.h.md) · [`obstacles_query.h`](obstacles_query.h.md) · [`ai_space.h`](ai_space.h.md) · [`xrAICore/Navigation/level_graph.h`](../xrAICore/Navigation/level_graph.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: a linear pass over two blocked-vertex sets per border operation

## Purpose

Adds obstacle avoidance to the border mechanism described in
[`restricted_object.cpp`](restricted_object.cpp.md). A border already confines a creature to
the corridor of its current path; this file also removes from consideration every navigation
vertex that some other object is standing on.

The mechanism is a **mask bit on the navigation graph's vertices**, not a restrictor. That
choice is the file's one real decision: obstacle sets change constantly and a restrictor is
a heavyweight authored thing, whereas a mask bit is a per-vertex flag the pathfinder already
consults and that costs one pass to set and one to clear.

## State

`Stateless.` The blocked sets belong to the two queries; the mask bits belong to the
navigation graph.

## The endpoint exemption

**Contract** — every masking pass excludes the path's own endpoints from the mask, and this
is the load-bearing idea in the file. A creature is itself an obstacle, and so is whatever it
is walking toward: masking the vertex it stands on would leave the pathfinder with no start,
and masking the destination would leave it with no goal. The exemption is applied in the
shape each border form describes its endpoints in:

```text
FUNCTION mask(query, endpoints)
  FOR EACH vertex IN query.blocked_area
    IF vertex IS an endpoint THEN CONTINUE      # never fence a creature out of where it
                                                #   is, or out of where it is going
    graph.set_mask(vertex)
```

- The **vertex-and-radius** form exempts the start vertex only. There is no destination
  vertex to exempt; the radius form fences a circle, not a route.
- The **start-and-destination-position** form exempts by *containment*: a blocked vertex is
  skipped if either position falls inside that vertex's cell. Positions are continuous and
  vertices are cells, so identity does not apply.
- The **start-and-destination-vertex** form exempts both by identity.

**Invariants** — the mask is set without validity checking, on the grounds that the blocked
sets were produced from the same graph and every vertex in them is by construction valid.
A rebuild that computes the blocked sets differently must re-establish that or check.

## `add_border` — the three forms

**Contract** — each first performs the base restrictor border, then masks the static query's
blocked area and the dynamic query's blocked area with the endpoint exemption appropriate to
its form. Ordering is fixed: the restrictor border first, the obstacle mask second.

**Invariants** — the two queries are masked **unconditionally**, including when the base
refused to install a restrictor border because the start was inaccessible. So a creature
standing in a forbidden region still gets obstacle masking. That is defensible — obstacles
are physical and apply regardless — but it means the base's install flag does not describe
whether this type changed anything, and `remove_border` clears the masks regardless, which
is what keeps the two halves consistent.

## `remove_border`

**Contract** — removes the base restrictor border, then clears the mask bits over both
queries' blocked areas. Unconditional, matching the unconditional set.

**Invariants** — clearing uses the *queries' current* areas, not a record of what was
masked. If a query's area changed between the border going up and coming down, the mask
clear misses vertices that were set and clears vertices that were not. Nothing prevents
that: the accessor assertion described in
[`restricted_object_obstacle.h`](restricted_object_obstacle.h.md) forbids *reading* a query
during a border but does not forbid the query from *refreshing itself*. In practice the
obstacle manager refreshes between paths, not during one. A rebuild should record the masked
set at install time and clear exactly that, which removes the hazard for the price of one
list per creature.
