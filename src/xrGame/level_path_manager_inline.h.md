# src/xrGame/level_path_manager_inline.h

> The level-graph path search: validated at entry, restricted to what the searcher is allowed to walk on, and told to forget its failures when those restrictions change.

**Needs** — [`level_path_manager.h`](level_path_manager.h.md) · [`abstract_path_manager.h`](abstract_path_manager.h.md) · [`restricted_object.h`](restricted_object.h.md) · [`ai_space.h`](ai_space.h.md) · [`xrAICore/Navigation/level_graph.h`](../xrAICore/Navigation/level_graph.h.md) · [Seam: Profiler and GPU debugging](../../SYSTEM-REQUIREMENTS.md#seam-profiler-and-gpu-debugging)
**Used by** — [`level_path_manager.h`](level_path_manager.h.md)
**Tier floor** — T3: a graph search with a per-vertex admission test

## Purpose

The generic path manager knows how to search a graph and cache the result. Three things are
true only of the *level* graph, and this file supplies exactly those three: the searcher has
restrictors that forbid vertices, the region it may search must be temporarily widened to
include the endpoints, and a change of restrictors invalidates what the manager remembers
about having failed.

## State

`Stateless.` — the route, the endpoints and the remembered failed endpoints all belong to
the generic manager. This file only overrides its hooks.

**Invariant** — the generic manager remembers the endpoint pair of its last *failed* search
so it can refuse an identical retry cheaply. That memory is only valid while the world's
reachability is unchanged, which is why `on_restrictions_change` exists.

## `build_path`

**Contract** — searches from one navigation vertex to another and caches the route. Both
vertices must be valid on the loaded level graph — this is a hard check that survives into
a shipping build, because a search from an invalid vertex would read outside the graph
rather than simply fail. Failure is a normal outcome, not an error: it leaves the manager
marked failed and the route empty. In a development build a failure logs the creature's
name and both endpoints with their positions, which is the only practical way to find out
which creature is stuck and where. Wrapped in a named profiler scope, because this is one of
the two or three most expensive things the AI does.

## `before_search` · `after_search`

**Contract** — the restrictor bracket around the generic search. Before: temporarily widen
the searcher's restrictors so that both endpoints are inside the searchable region. After:
restore them. Both do nothing when the searcher has no restrictors, and they always pair.

**Invariant** — once the border is applied, both endpoints are accessible. Widening exists
because a creature is routinely asked to path *to* the edge of its permitted region, or *from*
a vertex it was pushed onto; refusing either would leave it unable to move rather than
unable to reach one place.

## `check_vertex`

**Contract** — the per-vertex admission test the search calls on every vertex it considers.
Admits a vertex when the generic test admits it **and** the searcher's restrictors permit it.
This is what makes the route legal rather than merely short: the restrictor check is inside
the search, not a filter afterwards, so the search routes *around* a forbidden region instead
of finding the shortest path through one and then rejecting it.

**Notes** — a searcher with no restrictors passes everything, so an unrestricted creature
pays only the null test per vertex. With restrictors, this predicate runs once per expanded
vertex and is the dominant per-vertex cost of the search; a rebuild should make the
restrictor test cheap — a precomputed mask over the graph rather than a geometric query.

## `actual`

**Contract** — whether the cached route can still be used: true when the creature's *current*
navigation vertex is the route's start and the destination is unchanged. Comparing against
the creature's live position rather than the remembered start is the point — a creature that
has been shoved, teleported or has finished walking is no longer where its route begins, and
must re-path.

## `on_restrictions_change`

**Contract** — clears the remembered failed endpoint pair. Called when the searcher's
restrictors change. This is the whole of the invalidation: the route itself is not
discarded, because `actual` and the per-vertex test will catch a route that has become
illegal, but the *negative* cache would otherwise keep refusing a search that has just become
possible — which is the case of a door opening or a scripted barrier lifting, and is exactly
what a player notices when it goes wrong.

## `reinit`

**Contract** — rebinds the manager to a graph, defaulting to none, and drops everything
cached. Called at level load and at teardown. Pure delegation to the generic manager; the
override exists only to complete the specialization's surface.
