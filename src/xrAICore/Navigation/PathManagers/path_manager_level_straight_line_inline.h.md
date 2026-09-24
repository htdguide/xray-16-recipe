# src/xrAICore/Navigation/PathManagers/path_manager_level_straight_line_inline.h

> Measures what a route actually costs to walk, by string-pulling it — repeatedly skipping ahead as far as the mesh stays continuous and summing the straight segments instead of the cell hops.

**Needs** — [`path_manager_level_straight_line.h`](path_manager_level_straight_line.h.md) · [`path_manager_params_straight_line.h`](path_manager_params_straight_line.h.md) · [`path_manager_level_inline.h`](path_manager_level_inline.h.md) · [`../level_graph.h`](../level_graph.h.md)
**Used by** — [`path_manager_level_straight_line.h`](path_manager_level_straight_line.h.md)
**Tier floor** — T2: vector geometry plus a mesh traversal query.

## Purpose

A mesh route is a staircase of cell centres, and its cell count badly overstates how far a
creature really walks — a diagonal corridor costs twice its true length under a 4-connected
grid. Anything that *compares* destinations by distance needs the real figure. This policy
produces it: run the ordinary search, then walk the resulting route pulling it taut against the
mesh.

The shortened route is not kept. Only its length is, which is the whole point — the caller is
ranking candidates, not steering a creature.

## `setup`

**Contract** — as the mesh routing policy, plus: keep the request, and write the request's range
limit into its answer slot. That pre-seeding is the failure answer: a search that never reaches
the measuring step leaves "too far" behind rather than a stale number.

## `create_path`

**Contract** — builds the vertex route the ordinary way, then walks it forward maintaining an
*anchor* — the last point the taut string is pinned to — and a running length. Writes the total
into the request. Gives up early, reporting "too far", as soon as the running length reaches the
limit.

```text
FUNCTION measure_taut_length(route, start_point, dest_point, limit) -> real
  anchor        <- route[0]
  anchor_point  <- start_point          # the caller's real position, not the cell centre
  total         <- 0
  pending       <- 0                    # length of the longest visible reach from the anchor

  FOR EACH node AFTER route[0]
    IF the mesh stays continuous along the segment anchor_point -> position(node)
      pending <- distance(anchor_point, position(node))      # reach further; do not commit yet
    ELSE
      # the string caught on something: commit up to the last point we could see
      IF pending == 0                                        # not even the next node was visible
        total  <- total + mesh_distance(anchor, node)        # fall back to the mesh hop
        anchor <- node
      ELSE
        total   <- total + pending
        pending <- 0
        anchor  <- the previous node                         # pin at the last visible one and
                                                             #   re-examine this one next
      anchor_point <- position(anchor)

    IF total + pending >= limit
      RETURN limit                                           # "too far"; stop measuring

  # the final leg runs to the caller's real destination, not to the last cell centre
  IF the mesh stays continuous along anchor_point -> dest_point
    RETURN total + distance(anchor_point, dest_point)
  RETURN total + pending + distance(dest_point, position(last node of route))
```

**Invariants** — the route must be non-empty; the walk assumes a first element to anchor on.
The visibility question asked is not a ray cast against world geometry — it is the mesh's own
"walk from this vertex toward that point and tell me where you end up" traversal, so the taut
segment is one a creature could actually follow on the ground rather than one it could see
across a chasm. See [`../level_graph.h`](../level_graph.h.md).

**Notes** — the "too far" answer is the request's range limit written back verbatim, and the
same value is used internally to mean "this segment is blocked". Overloading one number with
three meanings — cost bound, blocked marker, too-far answer — is the least defensible thing in
this file; a rebuild should separate them.

The anchor is kept *in the output route's first slot*, which the routine overwrites as it walks.
The caller therefore receives a route whose first element is not the start vertex. Nothing
notices, because the only caller wants the length — but a rebuild must not reproduce it, and
anyone reusing this routine to produce a shortened route rather than a length has to fix it
first.

Backing up the anchor to the *previous* node when the string catches is what makes the result
taut rather than greedy-by-one: the segment committed is the longest one that stayed on the
mesh, and the node that broke it is re-examined from the new anchor rather than skipped.
