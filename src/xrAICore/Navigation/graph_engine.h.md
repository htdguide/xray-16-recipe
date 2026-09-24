# src/xrAICore/Navigation/graph_engine.h

> Declares the one object that owns every search the AI runs, and fixes the three assemblies it holds.

**Needs** — [`graph_engine_inline.h`](graph_engine_inline.h.md) · [`a_star.h`](a_star.h.md) · [`edge_path.h`](edge_path.h.md) · [`vertex_path.h`](vertex_path.h.md) · [`vertex_manager_fixed.h`](vertex_manager_fixed.h.md) · [`vertex_manager_hash_fixed.h`](vertex_manager_hash_fixed.h.md) · [`vertex_allocator_fixed.h`](vertex_allocator_fixed.h.md) · [`data_storage_bucket_list.h`](data_storage_bucket_list.h.md) · [`data_storage_binary_heap.h`](data_storage_binary_heap.h.md) · [`graph_engine_space.h`](graph_engine_space.h.md) · [`PathManagers/path_manager.h`](PathManagers/path_manager.h.md) · [`../Components/problem_solver.h`](../Components/problem_solver.h.md) · [Seam: Profiler](../../../SYSTEM-REQUIREMENTS.md#seam-profiler-and-gpu-debugging)
**Used by** — [`AISpaceBase.cpp`](../AISpaceBase.cpp.md) · [`problem_solver_inline.h`](../Components/problem_solver_inline.h.md) · [`graph_engine_inline.h`](graph_engine_inline.h.md) · [`abstract_location_selector_inline.h`](../../xrGame/abstract_location_selector_inline.h.md) · [`abstract_path_manager_inline.h`](../../xrGame/abstract_path_manager_inline.h.md) · [`action_planner.h`](../../xrGame/action_planner.h.md) · [`action_planner_inline.h`](../../xrGame/action_planner_inline.h.md) · [`ai_space.cpp`](../../xrGame/ai_space.cpp.md) · [`alife_monster_detail_path_manager.cpp`](../../xrGame/alife_monster_detail_path_manager.cpp.md) · [`alife_surge_manager.cpp`](../../xrGame/alife_surge_manager.cpp.md) · [`alife_update_manager.cpp`](../../xrGame/alife_update_manager.cpp.md) · [`map_location.cpp`](../../xrGame/map_location.cpp.md) · [`smart_cover.cpp`](../../xrGame/smart_cover.cpp.md) · [`space_restriction_composition.cpp`](../../xrGame/space_restriction_composition.cpp.md) · _and 1 more_
**Tier floor** — T1: it preallocates three search workspaces sized in megabytes so that no search allocates inside a frame.

## Purpose

Declares the surface implemented in [`graph_engine_inline.h`](graph_engine_inline.h.md). One
graph engine exists per running game; every path query in the engine goes through it. Its job is
not to search — that is [`a_star_inline.h`](a_star_inline.h.md) — but to *own the workspaces*.
Search state is large and preallocated, so it cannot be per-caller; making it per-engine is what
forces searches to be serial, and the search's own re-entrancy guard enforces that.

## State

Stateless. The engine's state is the three workspaces, described in [`graph_engine_inline.h`](graph_engine_inline.h.md); the assemblies they are built from are listed below.

## The three assemblies, and why they differ

```text
navigation search
  cost         real
  frontier     bucket list, 8192 buckets, stale buckets tolerated
  visited set  direct table, indexed by mesh vertex id
  pool         65536 vertices in the game; 2 million in the offline level compiler
  path record  vertices only
  metric       euclidian

planner search
  cost         int (16-bit)
  frontier     binary heap (costs have no bucketable range)
  visited set  hash, 256 buckets over 8192 index records
  pool         8192 vertices
  path record  edges (a plan is a list of operators)
  metric       euclidian

string search
  cost         real
  frontier     binary heap
  visited set  hash, 128 buckets over 1024 index records
  pool         1024 vertices
  path record  vertices only
```

**Invariants** — the navigation frontier's cost ladder is set at construction to span zero to two
thousand, which with 8192 buckets is about a quarter of a metre per bucket. Costs above two
thousand metres all land in the top bucket and lose their ordering, which is acceptable because a
search that long has already exhausted its budget.

The planner's workspace is constructed for 16384 vertices while its visited set and pool are
sized for 8192. The source flags this as a possible mistake and does not resolve it. The smaller
number is the real ceiling — the pool asserts first — so the extra headroom in the constructor
argument is inert. A rebuild should size all three consistently and pick one number.

The offline level compiler builds the same engine with a thirty-times-larger navigation pool and
without the planner or the string search at all, because it searches whole levels exhaustively
and never plans.

## Exported units

- `GraphEngine(max_vertex_count)` — construct all three workspaces; see the inline twin.
- `search(graph, start, goal, out_path, parameters)` — the navigation entry point, in four forms
  distinguished by whether the parameters are mutable and whether the caller supplies its own
  path manager.
- `search(problem_solver, start_state, goal_state, out_plan, parameters)` — the planner entry
  point.
- `search(graph, start_name, goal_name, out_path, parameters)` — the string-keyed entry point.
- `solver_algorithm()` — hand out the planner's workspace, for callers that need to inspect it.
- `PathTimer` — accumulated time spent searching, for the in-game statistics overlay.

**Notes** — a mutual-exclusion lock is declared on the engine and every use of it is commented
out. The searches are therefore *not* thread-safe, and the guard that actually protects them is
the re-entrancy check inside the search itself, which fails loudly rather than corrupting. A
rebuild has a real choice here: keep one engine and keep searches serial, or give each searching
thread its own engine and pay the workspace memory per thread. What is not available is sharing
one engine across threads.
