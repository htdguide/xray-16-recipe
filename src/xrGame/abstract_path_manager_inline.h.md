# src/xrGame/abstract_path_manager_inline.h

> Finds a route to a known destination, remembers whether that route still stands, and refuses to re-run a search that has already failed between the same two vertices.

**Needs** — [`abstract_path_manager.h`](abstract_path_manager.h.md) · [`restricted_object.h`](restricted_object.h.md) · [`ai_space.h`](ai_space.h.md) · [`xrAICore/Navigation/graph_engine.h`](../xrAICore/Navigation/graph_engine.h.md)
**Used by** — [`abstract_path_manager.h`](abstract_path_manager.h.md)
**Tier floor** — T2: a graph search on the frame budget, plus a small cache

## Purpose

Path finding here is not a function call, it is a piece of *retained state*. A creature
asks for a path once and then walks it for several seconds, during which the destination
may move, the evaluator may change its mind about edge costs, and the creature's movement
restrictions may change. This component owns the produced route, the question of whether
it is still valid, and the decision of how much of it to hand to the movement layer at a
time.

It also owns the one optimization that makes the whole AI affordable: a one-entry negative
cache. An unreachable destination is the common case — a creature is told to go somewhere
behind a locked door — and re-running a full graph search every update for a route that
cannot exist is what the cache exists to prevent.

## State

```text
RECORD PathManager
  graph               : optional<graph>       # bound navigation graph
  evaluator           : optional<scorer>      # edge/vertex cost model; also self-reports staleness
  path                : list<vertex id>       # the produced route, start first
  current_index       : int                   # how far the owner has walked; sentinel when unset
  intermediate_index  : int                   # how far along the path the owner has committed to go now
  dest_vertex_id      : vertex id             # the requested destination
  actuality           : bool                  # "the held path still answers the current question"
  failed              : bool                  # the last search produced no route
  failed_start        : vertex id             # the one-entry negative cache: the start of
  failed_dest         : vertex id             #   the last (start, dest) pair known to have no route
  object              : restricted object     # the creature whose restrictions bound every search
```

Invariants:

- `actuality` is false whenever anything the path depends on has changed: a new
  destination, a new evaluator, an evaluator that reports itself stale, or an explicit
  invalidation. It is never set true except by a successful search. This is the single
  flag the movement layer polls; everything else in the component exists to keep it
  honest.
- Both indices are reset to the "unset" sentinel by every search, successful or not, so a
  fresh path is never walked from a stale offset.
- The negative cache holds exactly one pair and is only written on failure. It is never
  *invalidated by time* — see the note on staleness below.

## `build_path`

**Contract** — produces a route between two graph vertices. Requires both vertices valid
and both bindings present. If the pair matches the negative cache the search is skipped
entirely and failure is reported immediately, with the subclass hooks still run so that
restriction state is prepared and restored identically on both paths. Otherwise the graph
engine searches, writing the route into the owned buffer. Either way the walk indices are
reset and actuality is set to the search's success. A fresh failure is recorded in the
cache.

```text
FUNCTION build_path(start, dest)
  REQUIRE graph and evaluator bound; start and dest valid vertices

  IF (start, dest) == (failed_start, failed_dest) THEN
    before_search(start, dest)
    failed = true
    after_search()
    reset both walk indices
    actuality = false
    RETURN                                  # no search: this pair is known hopeless

  before_search(start, dest)
  failed = NOT graph_engine.search(graph, start, dest, into: path, evaluator)
  after_search()
  reset both walk indices
  actuality = NOT failed

  IF failed THEN
    failed_start = start
    failed_dest  = dest                     # remember, so the next attempt costs nothing
```

**Notes**

- The hooks run on the cached-failure branch too. They are where a concrete manager widens
  the creature's movement restrictions for the search and restores them afterwards;
  skipping them on the short-circuit would leave the restriction state asymmetric, which
  is a far worse bug than a wasted search.
- The cache has no expiry. A door that opens, or a restriction that is lifted, does not
  clear it — the owner must call the explicit invalidation. That is a deliberate transfer
  of responsibility to whoever knows the world changed, and a rebuild must keep it
  explicit or it will strand creatures in front of doors that have since opened.

## `select_intermediate_vertex`

**Contract** — decides how far along the held path the creature commits to travel before
the path is reconsidered. The default is *all of it*: the intermediate index becomes the
last index. Requires a non-failed, non-empty path.

**Notes** — a concrete manager overrides this to commit to a nearer vertex, which is how
a creature following a long route re-evaluates partway instead of blindly walking a route
computed a minute ago. The default being "the whole path" makes the base behave as an
ordinary path finder, and the override is what makes it a *steering* component.

## `completed`

**Contract** — true when the committed segment reaches the end of the path, i.e. the
intermediate index is the last index. This is the signal the location selector consumes
as "path completed" to bypass its own throttle.

## `actual` / `make_inactual`

**Contract** — `actual` reports the stored flag and ignores the vertices it is handed; the
parameters exist so that a concrete manager can override with a comparison, and the base
does not need one because every mutator already maintains the flag. `make_inactual` forces
a rebuild on the next opportunity.

## `set_dest_vertex`

**Contract** — sets the destination, requiring it to pass the subclass's vertex check.
Actuality survives only if the destination is unchanged; any move of the destination drops
it.

```text
FUNCTION set_dest_vertex(vertex)
  REQUIRE check_vertex(vertex)
  actuality = actuality AND (dest_vertex_id == vertex)
  dest_vertex_id = vertex
```

## `set_evaluator`

**Contract** — binds the cost model. Drops actuality if the evaluator is a different one
*or* if the currently bound evaluator reports itself no longer actual — a cost model whose
inputs changed (new enemies, new danger, a changed restriction set) invalidates every path
built against it.

**Notes** — the staleness question is asked of the *old* evaluator before the new one is
installed, which is what makes "same evaluator, changed contents" invalidate. The check
reads through the existing binding, so this must not be the first call after a reset; in
practice a manager is always given its evaluator through a path that has one.

## `check_vertex`

**Contract** — the destination admissibility test; by default simply "is this a vertex of
the bound graph". A concrete manager narrows it, typically to "and it is inside my
movement restrictions".

## `reset` / `invalidate_failed_info`

**Contract** — `reset` clears the failure flag only, so a caller can retry after handling
a failure. `invalidate_failed_info` additionally clears the negative cache, which is the
only way a previously hopeless destination becomes searchable again. Callers invoke it
when the world changed in a way that could open a route.

## `reinit`

**Contract** — returns the component to the unbound, unbuilt state and optionally binds a
graph. Clears the route, both indices, both flags and the negative cache.

**Notes** — the destination slot is cleared with the *index* sentinel rather than the
*vertex* sentinel. The two happen to coincide in every instantiation that ships, so it is
harmless today; a rebuild with differently sized identities must not copy it.

## `path`, `intermediate_index`, `intermediate_vertex_id`, `dest_vertex_id`, `evaluator`, `failed`, `object`

**Contract** — plain reads. `intermediate_vertex_id` requires the committed index to be
inside the path; `object` requires the owning creature binding, which is set at
construction and never cleared.
