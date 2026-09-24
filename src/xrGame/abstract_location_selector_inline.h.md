# src/xrGame/abstract_location_selector_inline.h

> Chooses a destination vertex by scoring the navigation graph with a caller-supplied evaluator, throttled so that a creature does not re-decide where to go every frame.

**Needs** — [`abstract_location_selector.h`](abstract_location_selector.h.md) · [`restricted_object.h`](restricted_object.h.md) · [`ai_space.h`](ai_space.h.md) · [`xrAICore/Navigation/graph_engine.h`](../xrAICore/Navigation/graph_engine.h.md)
**Used by** — [`abstract_location_selector.h`](abstract_location_selector.h.md)
**Tier floor** — T2: a graph traversal with a per-search scorer callback on the frame budget

## Purpose

There are two distinct navigation questions in this game. "Take me to *there*" is the path
manager's job. This component answers the other one: "find me somewhere good" — the best
place to take cover, the best place to flee to, the best corpse to walk to. It is a
*search without a destination*: the graph engine expands outward from the creature's
current vertex and a caller-supplied evaluator scores each vertex it reaches, keeping the
best. The search terminates on the evaluator's own criteria, not on arrival.

The second problem it solves is stability. A creature that re-picks its cover position
every frame twitches between two nearly-equal candidates and never gets anywhere, so the
selector throttles searching to a configurable interval and treats *the answer not having
changed* as the normal outcome.

## State

```text
RECORD LocationSelector
  graph            : optional<graph>        # which navigation graph to search; none until bound
  evaluator        : optional<scorer>       # the caller's per-vertex scorer; none until bound
  selected_vertex  : vertex id              # the current answer; an invalid sentinel until first success
  failed           : bool                   # "the last search produced no NEW answer" — see below
  last_query_time  : int                    # global clock reading of the last search
  query_interval   : int                    # minimum milliseconds between searches; 0 means every call
  dest_path        : optional<ref list<vertex id>>   # caller's buffer for the winning path
  dest_vertex      : optional<ref vertex id>         # caller's slot for the winning vertex
  restricted_object: object                 # the creature whose movement restrictions bound the search
```

Invariants:

- The selector is *inert* unless both a graph and an evaluator are bound. Everything is
  written so that an unbound selector is a no-op rather than an error, because a creature
  carries several selectors and binds only the ones its current behaviour needs.
- `selected_vertex` is only ever read when valid; reading it before the first successful
  search is a contract violation, not a defined "no selection" value.
- The outputs are *references into the caller's own storage*, deposited only on a
  successful search. The creature therefore owns the path buffer and the selector never
  allocates one. That is why the selector can be re-bound to different consumers without
  reallocating.

## `select_location`

**Contract** — the main entry point. Runs a search if the selector is bound *and* either
the throttle interval has elapsed or the caller reports its current path completed;
otherwise clears the failure flag and returns without touching the selection. On a
successful search, deposits the winning vertex into the caller's slot if one was bound.
Does not allocate; blocks for the duration of the graph search, which is on the frame
budget.

**Invariants** — a completed path bypasses the throttle unconditionally. A creature that
has arrived must be allowed to decide where to go next *now*, whatever the interval says;
the interval exists to stop re-deciding mid-journey, not to stall an arrival.

```text
FUNCTION select_location(start_vertex, path_completed)
  IF bound AND (now >= last_query_time + query_interval OR path_completed) THEN
    perform_search(start_vertex)
    IF NOT failed AND dest_vertex bound THEN dest_vertex = selected_vertex
  ELSE
    failed = false        # nothing was attempted, so nothing failed
```

## `actual`

**Contract** — the same operation, phrased as a question the caller asks before rebuilding
its movement: *is my current destination still the right one?* Returns true when it is.

**Invariants** — this is where the inverted sense of "failed" pays off. An unbound or
throttled selector is trivially still actual. Otherwise a search runs, and its
*failure* — meaning either that no valid vertex was found or that the best vertex is the
one already selected — means the answer did not change, which is exactly "still actual".
So the function returns the failure flag as its truth value, and that is not a mistake.

```text
FUNCTION actual(start_vertex, path_completed) -> bool
  IF NOT bound OR (now < last_query_time + query_interval AND NOT path_completed) THEN
    RETURN true                          # nothing could have changed
  perform_search(start_vertex)
  IF NOT failed AND dest_vertex bound THEN dest_vertex = selected_vertex
  RETURN failed                          # failed == no new answer == still actual
```

**Notes** — the duplication between this and `select_location` is real: they differ only
in what the throttled branch does and in returning a value. A rebuild should write one
function returning "did the selection change".

## `perform_search`

**Contract** — the search itself. Gives the evaluator the caller's path buffer, then asks
the graph engine to search the bound graph starting from the creature's vertex with *the
same vertex as its target* and no path output of its own. Records the query time. Decides
failure. Runs the subclass hooks on both sides. Requires both bindings; violating that is
a contract error, not a runtime branch.

```text
FUNCTION perform_search(vertex)
  REQUIRE bound
  start = vertex
  before_search(start)                   # subclass may relocate the start, e.g. to a restricted region
  last_query_time = now
  evaluator.path_output = dest_path      # the evaluator, not the engine, records the winning path
  graph_engine.search(graph, from: start, to: start, path_output: none, evaluator)
  failed = evaluator.selected_vertex is invalid
           OR evaluator.selected_vertex == selected_vertex
  IF NOT failed THEN selected_vertex = evaluator.selected_vertex
  after_search()
```

**Notes**

- Passing the start vertex as its own target is how a destination-free search is expressed
  through a destination-taking engine: the engine's goal test never fires, the expansion
  is bounded by the evaluator, and the evaluator is the only thing that decides anything.
  A rebuild with an explicit "best-first scan" entry point does not need the trick.
- The engine is asked for no path, because the *evaluator* writes the shortest path to its
  own best-so-far vertex as it goes. Only the winner's path survives, so the engine cannot
  produce it after the fact.
- Reporting "the same vertex as last time" as a failure is the throttle's partner: the
  caller's movement is only rebuilt when the answer genuinely moved.

## `before_search` / `after_search`

**Contract** — do nothing by default. A concrete selector overrides them to widen or
narrow the creature's movement restrictions around the search, or to relocate an illegal
start vertex to a legal one. They exist because the restriction state must be restored
even when the search decides nothing, which a single hook could not guarantee.

## Accessors

**Contract** — `get_selected_vertex_id` returns the current answer and requires it to be
valid. `failed` reports whether the last search produced a new answer (inverted, as
above). `used` reports whether both a graph and an evaluator are bound. `set_evaluator`,
`set_query_interval`, `set_dest_path` and `set_dest_vertex` are plain bindings with no
side effects — notably, changing the evaluator does *not* invalidate the current
selection.

## `reinit`

**Contract** — returns the selector to the unbound, unselected state and optionally binds
a graph. Called when the owning creature respawns or changes level, so the bindings and
the throttle clock must not survive.
