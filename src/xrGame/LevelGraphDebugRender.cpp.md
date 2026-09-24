# src/xrGame/LevelGraphDebugRender.cpp

> The navigation overlays: the level's walkable mesh, the cross-level graph miniature, restrictor borders, per-vertex cover values, and where offline creatures currently are.

**Needs** — [`LevelGraphDebugRender.hpp`](LevelGraphDebugRender.hpp.md) · [`Level.h`](Level.h.md) · [`debug_renderer.h`](debug_renderer.h.md) · [`space_restriction_manager.h`](space_restriction_manager.h.md) · [`cover_manager.h`](cover_manager.h.md) · [`cover_point.h`](cover_point.h.md) · [`ai_space.h`](ai_space.h.md) · [`alife_simulator.h`](alife_simulator.h.md) · [`alife_graph_registry.h`](alife_graph_registry.h.md) · [`CustomMonster.h`](CustomMonster.h.md) · [`ai/stalker/ai_stalker.h`](ai/stalker/ai_stalker.h.md) · [`smart_cover_object.h`](smart_cover_object.h.md) · [`team_base_zone.h`](team_base_zone.h.md) · [`xrAICore/Navigation/level_graph.h`](../xrAICore/Navigation/level_graph.h.md) · [`xrAICore/Navigation/game_graph.h`](../xrAICore/Navigation/game_graph.h.md) · [Seam: Debug overlay UI](../../SYSTEM-REQUIREMENTS.md#seam-debug-overlay-ui)
**Used by** — reached through its declarations in [`LevelGraphDebugRender.hpp`](LevelGraphDebugRender.hpp.md); callers name that, not this file.
**Tier floor** — T2: spatial queries and immediate-mode drawing; present only in a development build

## Purpose

Navigation data in this engine is prebuilt, opaque and enormous — a grid of walkable
vertices with packed planes and cover values per level, plus a coarse cross-level graph. When
a creature goes somewhere unexpected, the only way to find out why is to look at the data.
This file is that instrument, and it exists entirely outside a shipped build.

It is worth a twin despite being development-only, because **what it chooses to draw is a
description of what the navigation data actually contains**. A rebuilder who understands
these six overlays understands the navigation model: vertices carry a plane and four
directional cover values at two heights; the cross-level graph carries travel between named
places; restrictors are sets of border vertices; offline creatures are positions *along an
edge* rather than at a vertex.

## State

```text
RECORD LevelGraphDebugRender
  game_graph        : the cross-level graph being drawn
  level_graph       : the loaded level's walkable mesh
  debug_shader      : the untextured material the mesh quads draw with
  current_level_id  : int      # which level's slice of the game graph to show; -1 = all
  current_actual    : bool     # is the cached extent below still valid
  current_center    : position # extent of that slice, used to normalize the miniature
  current_radius    : position
  cover_point_cache : list<CoverPoint>   # reused scratch, to avoid a per-frame allocation
```

**Invariants** — the game graph's vertices are **sorted by level**, so a scan for one level's
vertices may stop at the first non-matching vertex after the first match. Every traversal
here relies on that, and it is a property of the shipped data, not of the code.

## `Render`

**Contract** — the entry point, called once per frame from the level's render. Each overlay
is behind its own flag, and the flags compose.

```text
FUNCTION render(game_graph, level_graph)
  IF flag "draw game graph"     -> draw_game_graph()
  IF neither general debug nor the motion flag is on
    RETURN
  IF general debug AND the ai-debug flag  -> draw_nodes()
  draw_restrictions()                      # unconditional once past the gate
  IF flag "cover"  -> draw_covers()
  IF flag "motion" -> draw_objects()
  draw_debug_node()
```

**Notes** — the restrictor overlay has no flag of its own and rides on the general debug
gate. That is inconsistent with its five neighbours and reads as an oversight.

## `DrawGameGraph`

**Contract** — draws the cross-level graph as a **miniature floating above the player**,
rather than in place. The graph spans every level in the campaign and its vertices are tens
of kilometres apart, so drawing it at world scale would be useless; instead the selected
slice's extent is normalized into a small box in front of the camera. A flag switches to
real-world positions for the rare case where the slice *is* the current level.

Each vertex draws its edges to lower-numbered neighbours only, so every edge is drawn once.
Optionally, offline stalkers and offline monsters standing on each vertex are drawn too.

**Notes** — a dead-code path drew an opaque backing plane behind the miniature so the graph
would not be lost against the world. It is disabled, and without it the miniature is hard to
read over bright geometry.

## `UpdateCurrentInfo` / `ConvertPosition` / `Modify`

**Contract** — computes the bounding box of the selected slice of the graph, including each
vertex's neighbours so edges leaving the slice are contained, and then maps a graph position
into the miniature.

```text
FUNCTION convert_position(graph_point) -> world position
  p = center - graph_point                  # note the subtraction order
  p.x = p.x * 5 / radius.x                  # horizontal axes stretched five times
  p.y = p.y * 1 / radius.y                  # vertical left alone
  p.z = p.z * 5 / radius.z
  p = p * 0.5
  RETURN p + current_entity.position + (0, 4.5, 0)
```

**Notes** — the five-to-one horizontal stretch exists because the campaign's levels are laid
out on a broad, nearly flat plane; without it the miniature collapses into a line. The 4.5
unit lift puts it above the player's head. Both are display tuning with no other consequence.

The subtraction is center minus point, which mirrors the miniature. Nothing suggests that
was intended.

The extent is cached behind a validity flag that `SetupCurrentLevel` clears — but the draw
recomputes it unconditionally every frame anyway, so the cache never takes effect.

## `DrawStalkers` / `DrawObjects(vertex)`

**Contract** — draws the offline creatures registered on one graph vertex, in two passes with
different meanings, and the split is the load-bearing part:

- A creature **standing** at the vertex — no detail path, or a path it has not started
  walking — is drawn once at the vertex itself. Only the first such creature per vertex is
  drawn, because they would all occupy the same point.
- A creature **travelling** is drawn at its interpolated position along the edge it is on:
  the fraction is its walked distance over the edge's length, and the position is that
  fraction between the two vertices.

```text
FOR EACH offline creature registered at this vertex
  IF its detail path has fewer than two entries
    CONTINUE                       # not travelling
  from = its current graph vertex
  to   = the second-to-last entry of its detail path
  fraction = walked_distance / distance(from, to)
  draw a marker at from + (to - from) * fraction
```

**Invariants** — this is the clearest statement in the tree of how an offline creature's
position is represented: **a graph edge plus a distance walked along it**, not a coordinate.
That is what makes the alife simulation cheap, and a rebuild must reproduce it.

**Notes** — the two routines, one for stalkers and one for monsters, are the same algorithm
against two different offline record types. The duplication is real and a rebuild should
write it once over the common base.

The destination vertex is taken as the *second to last* entry of the path, not the last.
Whether that is the next hop or an off-by-one is not recoverable from the source.

## `DrawNodes`

**Contract** — draws the level's walkable mesh near the camera: each vertex as a quad lying
on its own stored plane, its centre as a small box, and its identifier when it is the vertex
the player stands on or one of that vertex's neighbours.

```text
FUNCTION draw_nodes()
  here = current entity's level vertex
  neighbours = the vertices linked from here     # at most 128 collected

  # the mesh is sorted by planar position, so the visible band is a RANGE,
  # found by binary search rather than by scanning the whole mesh
  first = lower bound of (camera position - 30) in planar order
  last  = upper bound of (camera position + 30) in planar order

  FOR EACH vertex IN first .. last
    SKIP IF further than 30 units from the camera
    SKIP IF its cell-sized sphere is outside the view frustum
    colour = white, or green if it is "here", or dark green if it is a neighbour
    # the vertex stores a compressed normal; the quad is its cell square
    # projected UP onto that plane, then lifted a hair to avoid coplanar fighting
    plane = plane through the vertex position with the decompressed normal
    corners = the four cell-square corners raised vertically onto that plane,
              then offset along the normal by a small epsilon
    draw two triangles; draw a small box at the centre
    IF it is "here" or a neighbour, draw its identifier above it
```

**Invariants** — the mesh being sorted in planar order is what makes this affordable, and it
is the same property the pathfinder relies on. Drawing the quad *on the vertex's plane*
rather than flat is what makes sloped ground legible, and it is also the reminder that each
navigation vertex carries an orientation.

**Notes** — the quad is drawn at 98% of the cell size so adjacent vertices show a seam;
without the gap the mesh reads as one surface. The coplanar offset is one hundredth of a
unit. The neighbour list is capped at 128, which is far above the mesh's real degree.

## `DrawRestrictions`

**Contract** — draws every live restrictor's **border vertices** — not its volume — each in
a colour drawn from a fresh generator. Released and uninitialized restrictors are skipped.

**Notes** — a restrictor is stored as its border, which is the set of navigation vertices on
its boundary. That is why the pathfinder can treat a restrictor as a cost modification
rather than as a geometric test.

The colour generator is created fresh each frame with no seed, so each restrictor's colour is
stable across frames but arbitrary. Two restrictors can collide on a colour.

## `DrawCovers`

**Contract** — for every cover point within five metres of the camera, draws the cover data
of its navigation vertex, twice — once at standing height and once at crouching height.
Reading it is the way to understand what the cover model actually stores.

```text
FOR EACH cover point near the camera
  FOR height IN { high (1.5 up), low (0.6 up) }
    draw a box marking the point
    # 1. the continuous function: exposure sampled every 10 degrees
    FOR angle IN 0, 10, 20 ... 350 degrees
      draw a ray of length proportional to cover_in_direction(angle)
    # 2. the stored data: FOUR values, one per axis direction, scaled by 15
    draw four rays along -x, +z, +x, -z of length cover(i) * half_cell / 15
    # 3. the best direction: the angle maximizing the cover integrated
    #    over a quarter-turn arc, drawn in black
    draw the winning ray
```

**Invariants** — the four axis values are what the level data actually *stores* per vertex;
everything else drawn here is computed from them. Their scale divisor of 15 means the stored
values are in a small integer range whose maximum is 15 — a 4-bit quantity per direction, per
height. That packing is the reason cover data for a whole level fits alongside the mesh.

The "best direction" is the angle maximizing cover integrated over a **quarter turn**, not
the direction of maximum cover at a point. The arc width is the decision: cover that protects
you from one exact angle is worthless, and this is where the width is chosen.

**Notes** — the scratch list of nearby cover points is reserved for a thousand entries and
reused between frames, because the query would otherwise allocate every frame. The five-metre
query radius keeps the overlay readable, not correct — cover is queried at much greater
ranges in play.

## `DrawObjects`

**Contract** — walks every object in the level and lets three kinds draw themselves: team
base zones, creatures, and smart covers. For a creature it additionally marks the **end of
its detail path**, which is where it currently believes it is going.

## `DrawDebugNode`

**Contract** — draws a tall marker at each of two vertices named by console variables, one
blue and one red. It is the manual probe: type two vertex identifiers and see where they are.

## `SetupCurrentLevel`

**Contract** — selects which level's slice of the cross-level graph the miniature shows, and
invalidates the cached extent when it changes. The value -1 means every level at once.
