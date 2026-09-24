# src/xrAICore/Navigation/PathManagers — the search policies

> One search implementation answers six different questions, because everything that differs
> between them is a policy object and a request record rather than a second algorithm.

Part of [chapter 14](../../README.md); the algorithm these policies steer is in
[`Navigation/`](../README.md).

## What this directory is responsible for

A *path manager* is the bundle of answers the search loop needs but does not decide:

- what does traversing this edge cost;
- what is the estimated remaining cost from here (zero turns the search back into
  uniform-cost);
- may this vertex be entered at all;
- have we finished — is this the goal, or have we run past a radius, or has a budget expired;
- and what, if anything, should be written out as each vertex settles.

A *parameter record* is the caller's request: the budgets, plus whatever the particular
question needs as input and whatever slot the answer is written back into. The two are
paired: the dispatch rule turns *(which graph, which parameter record)* into the right
policy, so **choosing a parameter record is how a caller states what kind of answer it
wants**. There is no other switch.

## Where it sits

It rests entirely on [`Navigation/`](../README.md) — the graphs and the search loop — and on
the math layer. Nothing here is creature-specific except by borrowing: the cross-level
terrain policy holds a preference table that a creature supplies, but it does not know what a
creature is. Its consumers are the engine's search entry points and, through them, every
brain in chapter 24.

## The load-bearing ideas

**Refinement, not replacement.** One generic policy supplies the complete list of questions
with defaults that forward to the graph: take every cost from the graph, use no heuristic,
admit every vertex, stop at the goal, honour the three budgets. Every other policy in the
directory overrides two or three of those answers and inherits the rest. A rebuild gets the
same structure from any dispatch mechanism it likes; what matters is that the question list
is fixed and closed.

**The two graphs get different cost models, deliberately.** On the cross-level graph, edge
costs are the graph's own measured lengths and the heuristic is straight-line distance — so
routes there are genuinely shortest. On the level mesh, every step costs exactly one cell and
the heuristic is *twice* the admissible Manhattan estimate. That factor is the chapter's
loudest trade: it roughly halves the number of expanded cells and it permits a route up to
twice the optimal length. Settled vertices on the mesh are additionally never reconsidered,
which is only consistent with an inadmissible estimate because the estimate is inflated on
purpose.

**A search without a goal is still a search.** Two of the policies never terminate on a
vertex identity at all. The flood fill runs until a radius is exceeded and collects
everything it settled, in increasing-distance order. The nearest-vertex search floods the
same way but keeps only the one settled vertex that ends up horizontally closest to a target
point — a point that need not lie on the mesh, which is exactly why the search is needed.

**A route's length is not the route's cost.** The mesh search counts cell hops, and a
creature does not walk cell hops: it walks the string-pulled line. So there is a separate
policy that routes on the mesh and then measures the result by repeatedly skipping as far
ahead as the mesh stays continuous and summing the straight segments. Its parameter record is
also its result slot, and one number in it carries three meanings at once — cost bound,
blocked marker, and the "too far" answer.

**Terrain preference is a mask, with one escape clause.** A creature confines a cross-level
search to vertex types it will cross. A creature that is *already standing* on a vertex it
would not choose may move anywhere until it is off one — otherwise a badly placed spawn
strands an entity permanently.

**The planner is a graph as far as this directory is concerned.** One policy adapts the
search engine to the planner: vertices are world states, edges are actions. It is the only
policy whose result is a list of edges rather than a list of vertices, because a plan is a
sequence of actions, not of intermediate world states.

**The budgets live in the base request record.** Three numbers — a ceiling on estimated total
cost, a ceiling on expansions, and a cap on vertices touched — are part of every request, and
the third is the one that binds, because it sits just under the fixed vertex pool's capacity.
A policy may narrow them; none may widen past the pool.

## The twins

### The dispatch and the base

| Twin | Role |
|---|---|
| [`path_manager.h`](path_manager.h.md) | The single name every search is issued through, and the rule turning a (graph, parameter-set) pair into a policy |
| [`path_manager_generic.h`](path_manager_generic.h.md) | The closed list of questions the search asks about a search |
| [`path_manager_generic_inline.h`](path_manager_generic_inline.h.md) | The defaults every other policy refines: costs from the graph, no heuristic, stop at the goal, honour the budgets |
| [`path_manager_params.h`](path_manager_params.h.md) | The three budgets every request carries, and what each one means |

### Routing on the level mesh

| Twin | Role |
|---|---|
| [`path_manager_level.h`](path_manager_level.h.md) | Declares the mesh routing policy |
| [`path_manager_level_inline.h`](path_manager_level_inline.h.md) | One cell per step, twice the Manhattan estimate, settled vertices never revisited — fast, and not guaranteed shortest |
| [`path_manager_level_straight_line.h`](path_manager_level_straight_line.h.md) | Declares the policy that also reports what the route costs once shortened by line of sight |
| [`path_manager_level_straight_line_inline.h`](path_manager_level_straight_line_inline.h.md) | String-pulling: skip ahead while the mesh stays continuous, sum the straight segments |
| [`path_manager_params_straight_line.h`](path_manager_params_straight_line.h.md) | The request that is also the result slot: two endpoints in, a measured length out |

### Searching the mesh without a goal

| Twin | Role |
|---|---|
| [`path_manager_level_flooder.h`](path_manager_level_flooder.h.md) | Declares the goal-less search collecting everything reachable within a radius |
| [`path_manager_level_flooder_inline.h`](path_manager_level_flooder_inline.h.md) | The flood, delivered in order of increasing distance |
| [`path_manager_params_flooder.h`](path_manager_params_flooder.h.md) | The flood request: a radius, and budgets shaped around it |
| [`path_manager_level_nearest_vertex.h`](path_manager_level_nearest_vertex.h.md) | Declares the search for the reachable mesh vertex closest to an arbitrary point |
| [`path_manager_level_nearest_vertex_inline.h`](path_manager_level_nearest_vertex_inline.h.md) | Flood within a radius, keep the single horizontally closest settled vertex |
| [`path_manager_params_nearest_vertex.h`](path_manager_params_nearest_vertex.h.md) | The request: a radius plus the point to measure against |

### Routing on the cross-level graph

| Twin | Role |
|---|---|
| [`path_manager_game.h`](path_manager_game.h.md) | Declares the cross-level routing policy |
| [`path_manager_game_inline.h`](path_manager_game_inline.h.md) | True stored edge lengths, a straight-line heuristic, no budget at all |
| [`path_manager_game_vertex.h`](path_manager_game_vertex.h.md) | Declares the policy confining a cross-level search to terrain a creature will cross |
| [`path_manager_game_vertex_inline.h`](path_manager_game_vertex_inline.h.md) | The terrain mask, and the escape clause for a creature already standing where it would not go |
| [`path_manager_params_game_vertex.h`](path_manager_params_game_vertex.h.md) | The request holding the borrowed preference table |
| [`path_manager_game_level.h`](path_manager_game_level.h.md) | Declares the "get me to that level" search — any vertex on a named level |
| [`path_manager_game_level_inline.h`](path_manager_game_level_inline.h.md) | Uniform cost, terminating at the cheapest vertex belonging to the named level |
| [`path_manager_params_game_level.h`](path_manager_params_game_level.h.md) | The request: a level identifier in, the vertex that was reached out |

### Searching the planner

| Twin | Role |
|---|---|
| [`path_manager_solver.h`](path_manager_solver.h.md) | Declares the adapter that lets the search engine search the planner instead of a graph |
| [`path_manager_solver_inline.h`](path_manager_solver_inline.h.md) | The only policy whose answer is a list of edges, because a plan is a sequence of actions |
