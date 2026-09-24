# src/xrAICore — chapter 14

> The generic AI machinery: navigation graphs, the one search that walks all of them, and the
> goal/plan/action layer. Nothing here knows what a creature is.

## What this module is responsible for

Two families live here and they are the same machine seen twice.

The first is **navigation**: a two-level world graph (a fine per-level navigation mesh inside a
level, a coarse graph between levels, and a cross table binding one to the other), one A*
implementation with its open and closed structures, and a family of *search policies* that let
that single implementation answer very different questions — a route across a level, a route
between levels, a flood fill of everything within a radius, the nearest reachable spot to a
point, the true walked length of a route.

The second is the **goal/plan/action layer**: a world state is a set of property/value pairs,
an action declares which properties it requires and which it changes, and a planner searches
over actions for a sequence that reaches a goal state. The planner does not have its own search:
it presents itself to the navigation layer's A* as a graph whose vertices are world states and
whose edges are actions, and the answer that comes back is a plan.

Also here: the patrol-path registry — authored waypoint graphs that the game layer hands to
creatures — and a small value type pairing an object's mesh vertex with its cross-level vertex.

The module is deliberately creature-free. Every concrete action, every property evaluator and
every creature brain is chapter 24; this chapter is the vocabulary those are written in.

## Where it sits

Chapter 14, after the engine (13) and before the UI and physics chapters. It rests on the core
services (virtual filesystem, configuration, containers, threading) of chapter 6, on the math
layer of chapter 3, on the script engine of chapter 10 for the handful of types it exports to
Lua, and on the global environment struct of chapter 5 — which it both fills (the navigation
owner registers itself there) and reads from (several routines reach navigation through it
rather than being handed it).

It is one half of a declared cycle: the engine owns the frame loop and declares the interfaces,
and this module links back against it to be driven. In a rebuild the engine depends on abstract
ports and this module supplies adapters. The recipe notes each reach through the global struct
where it occurs.

## The load-bearing ideas

Name these once here so the twins can be terse.

**One search, many policies.** There is exactly one path-search implementation in the engine —
an A* whose Dijkstra base is the same code with the heuristic switched off. Every difference
between the searches listed above lives in a *policy object* that answers a fixed set of
questions: what does an edge cost, what is the estimated remainder, which vertices may be
entered, when do we stop, and what do we write out. The policy is chosen by the pair *(which
graph, which parameter record the caller passed)*, so choosing a parameter record is how a
caller states what kind of answer it wants. See
[`Navigation/PathManagers/README.md`](Navigation/PathManagers/README.md).

**Two graphs and a cross table.** Inside a level, a *level graph* — a uniform grid of cells,
four-connected, each cell one vertex, all distances derived from the cell width. Between levels,
a *game graph* — a few thousand vertices with real world positions and real stored edge lengths,
spanning the whole game. A *cross table* maps every mesh vertex of the loaded level to the game
graph vertex that covers it. The three carry identity stamps and the world refuses to load if
they disagree, because a mismatched pair paths creatures through walls silently.

**The searches are bounded, and not resumable.** Every search carries three budgets: an
estimated-total-cost limit, an expansion limit, and a limit on how many vertices may be touched.
The last is the one that matters, and it is set just below the capacity of a fixed, pre-allocated
vertex pool that the search never grows. A search that exhausts a budget reports failure and
leaves no frontier behind: there is no continuation, and a caller that wants to spread work
across frames re-issues the whole search. That is why the budgets are small enough to afford
every frame rather than large enough to guarantee an answer.

**Optimality is traded away deliberately, and not uniformly.** Routes on the *game graph* are
genuinely shortest: the heuristic is straight-line distance and the stored edge lengths are at
least that. Routes on the *level mesh* are not: the heuristic is twice the admissible Manhattan
estimate, which roughly halves the number of expanded cells and can return a route up to twice
the optimal length. Plans from the planner's forward search are not optimal either, for a
different reason given below. A rebuild that "fixes" any of these gets better answers and a
slower, hungrier search; the original chose speed everywhere it could see the seam.

**Cost is quantized.** The level search's open set is not a heap but a bucketed queue: 8192
buckets spanning costs from 0 to 2000 world units, so vertices within about a quarter of a world
unit of each other are indistinguishable in priority and are ordered by discovery instead.
Everything beyond 2000 falls into one final bucket with no ordering at all. This is what makes
insertion and extraction constant-time, and it is the real tie-breaking rule of the engine's
pathfinding.

**A world state is a delta, not a snapshot.** The planner's search vertex records only the
properties whose value *differs from the world as measured*. The start vertex of a forward
search is therefore the empty state, applying an action can make a state smaller, and two
different action sequences that reach the same world reach the same vertex. The state carries a
hash that is the XOR-fold of its properties' hashes — order-independent, maintained
incrementally — and that hash is what the search's visited-state table buckets on.

**The world is measured lazily and cached per plan.** No property is evaluated until some branch
of the search asks about it; the answer is then cached for the rest of that plan. Re-planning is
triggered by re-measuring exactly the properties the previous plan consulted and seeing whether
any moved. A world change that no standing plan depended on costs nothing.

**What the planner guarantees, and what it merely tries for.** *Guaranteed*: every action in the
returned plan had its preconditions satisfied where it appears, given the world as measured; the
plan is bounded in length by the visited-state budget; a plan is never executed after a property
it relied on has changed. *Merely attempted*: that the plan is cheapest — the forward search's
heuristic counts every goal property absent from the delta as outstanding, including those the
world already satisfies, so it overestimates and the search can settle for a costlier plan. And
*not attempted at all*: that a plan exists. The search gives up after a bounded number of visited
states, and giving up is indistinguishable from "impossible".

**An action's cost is bounded below by what it changes.** Every action reports a lower bound
equal to the number of properties it actually changes, and any cost it declares must be at or
above it. That is not a sanity check: the planner's heuristic counts unsatisfied properties, so
one unit of cost must buy at most one satisfied property or the estimate stops being a bound.

## Twins in this directory

| Twin | Role |
|---|---|
| [`AISpaceBase.cpp`](AISpaceBase.cpp.md) | Owns the level graph, the search engine and the patrol registry; borrows the game graph; enforces the three-way identity check at level load |
| [`AISpaceBase.hpp`](AISpaceBase.hpp.md) | Its declaration, and the split between lifecycle operations and public accessors |

Not given twins: the build files (`CMakeLists.txt`, the two project files) and the precompiled
header pair (`pch.cpp`, `pch.hpp`). They carry no decision a rebuild must reproduce. The build
files do record one fact worth keeping: this module links against the core, the engine, the math
layer, the global-environment module and the script engine, and nothing else.

## Subdirectories

| Directory | Contents |
|---|---|
| [`Components/`](Components/README.md) | The goal/plan/action layer: world properties, world states, operators, the planner, and their script surface |
| [`Navigation/`](Navigation/README.md) | The graphs, the search algorithms, the open/closed structures, the object-location value type |
| [`Navigation/PathManagers/`](Navigation/PathManagers/README.md) | The search policies — what makes one search answer many questions |
| [`Navigation/PatrolPath/`](Navigation/PatrolPath/README.md) | Authored waypoint graphs and their registry |

## What could not be recovered

Collected from across the chapter, so the honesty section of the root README can absorb it.

- **The fifteen-centimetre lift** applied to a patrol waypoint's position before it is matched to
  a mesh cell. The effect is clear — it stops a point authored flush with the floor resolving to
  whatever lies beneath — but nothing in the source derives the number.
- **The heuristic weight of two** on the level mesh. Reproducible and clearly deliberate (the file
  keeps disabled alternatives beside it), but no record of what it was measured against.
- **The bucketed queue's range of 0 to 2000.** Costs beyond it are unordered. Nothing says whether
  2000 was chosen from level dimensions or from measurement, and the default the structure itself
  carries is 1000 — overridden at construction to 2000 with no comment.
- **The planner's budget of 8000 visited states** is explicable (it sits just under a fixed pool of
  8192) but the pool size is not. A neighbouring comment in the search engine notes that the
  solver algorithm is *constructed* for 16384 vertices while its manager and allocator are sized
  for 8192, and flags it as possibly a mistake. It has not been resolved here.
- **`weight` on the world state** is declared taking a single property but its body requires a whole
  state. It compiles only because it is never instantiated. Whether it was meant to survive is
  unknown; the planner computes the same quantity itself.
- **`property` on the world state** returns the next property at or after the requested one when the
  requested one is absent, rather than nothing. This is reachable from script. Whether callers
  were expected to check, or whether nobody noticed, is not recoverable.
- **The patrol enumerations collide in the script namespace**: `stop` and `dummy` are registered
  twice with different meanings, and which registration survives depends on the binding layer's
  ordering. The shipped scripts depend on whichever one wins. This must be verified against a
  running original rather than trusted from the source.
- **An alias in the patrol registry** makes two keys point at one owned path, and teardown frees
  every entry. No shipped path takes this route, so it has never surfaced; whether the aliasing
  feature was intended to coexist with teardown at all is unclear.
- **A named constant holding one particular patrol path's name** sits unused in the patrol-path
  source. A debugging leftover with no recoverable purpose.
- **The straight-line request's range field** carries three meanings at once — cost bound, blocked
  marker, and the "too far" answer. That they are the same number is load-bearing for callers,
  but there is no statement anywhere that it was a choice rather than a convergence.
