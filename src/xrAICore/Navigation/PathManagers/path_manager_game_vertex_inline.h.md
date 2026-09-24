# src/xrAICore/Navigation/PathManagers/path_manager_game_vertex_inline.h

> The same cross-level route, seen through one creature's terrain preferences — with the rule that a creature already standing somewhere it would not choose to go may move anywhere until it is out.

**Needs** — [`path_manager_game_vertex.h`](path_manager_game_vertex.h.md) · [`path_manager_params_game_vertex.h`](path_manager_params_game_vertex.h.md) · [`path_manager_game_inline.h`](path_manager_game_inline.h.md) · [`../game_graph_space.h`](../game_graph_space.h.md)
**Used by** — [`path_manager_game_vertex.h`](path_manager_game_vertex.h.md)
**Tier floor** — T2: a mask comparison.

## Purpose

Different creatures cross different ground. Rather than build a graph per creature kind, the
one cross-level graph carries terrain attributes on every vertex and each creature carries a
table of acceptable attribute masks; this policy is the join. It is how a single compiled
navigation dataset serves every creature in the game.

## `setup`

**Contract** — as the cross-level routing policy, plus two things: it keeps the request (which
holds the borrowed preference table) and it evaluates the filter once on the *start* vertex,
remembering the answer.

**Invariants** — the start-vertex test must happen after the rest of the setup, because the
filter it runs needs the graph bound.

## `is_accessible`

**Contract** — three rules in order.

```text
FUNCTION is_accessible(vertex) -> bool
  IF NOT graph.enabled(vertex)        RETURN false   # a closed vertex is closed for everyone
  IF NOT start_passed_the_filter      RETURN true    # the escape hatch, see below
  FOR EACH mask IN creature.vertex_types
    IF mask matches vertex.terrain_attributes  RETURN true
  RETURN false
```

**Invariants** — the escape hatch is evaluated against the *start* vertex only, decided once at
setup, and applies for the whole search. It is not re-evaluated as the search moves.

**Notes** — the escape hatch is the load-bearing decision in this file. A creature can end up
standing on terrain its own preference table rejects: it was pushed there, it was spawned there
by an authored placement, or its preferences were changed while it stood still. Without the
hatch its own filter would reject its own vertex, every neighbour, and the search would fail
immediately — the creature would be permanently stuck. With it, such a creature is allowed to
path anywhere at all until it happens to leave. That is a blunt instrument: a creature in this
state can route straight through terrain it should never enter, for the whole journey rather
than just far enough to escape. A rebuild that wants the same rescue with less collateral
should relax the filter only until the first acceptable vertex is reached, then re-impose it.

An empty preference table makes every vertex fail the filter, which is a mistake in the
creature's configuration rather than a request. The policy warns about it in debug builds and
otherwise lets the search fail.

The terrain match is a mask test rather than an equality: a vertex's attributes and a
creature's mask are both small attribute vectors, and matching means the mask admits the
vertex's value in every position. See [`../game_graph_space.h`](../game_graph_space.h.md).
