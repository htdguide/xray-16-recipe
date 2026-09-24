# src/xrGame/smart_cover_description.cpp

> Loads one kind of smart cover out of the authored script tables: its loopholes, the graph of transitions between them, and the four connectivity guarantees that make the cover usable.

**Needs** — [`smart_cover_description.h`](smart_cover_description.h.md) · [`smart_cover_loophole.h`](smart_cover_loophole.h.md) · [`smart_cover_transition.hpp`](smart_cover_transition.hpp.md) · [`smart_cover_detail.h`](smart_cover_detail.h.md) · [`smart_cover_object.h`](smart_cover_object.h.md) · [`ai_space.h`](ai_space.h.md) · [Seam: Script virtual machine](../../SYSTEM-REQUIREMENTS.md#seam-script-virtual-machine)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: graph construction from a script table; load-time, not frame-time

## Purpose

Builds the shared template for a kind of smart cover. The load is three ordered steps, and
the order is load-bearing: loopholes first, then transitions, then a pass over the
loopholes that can only be answered once the graph exists — *can this loophole be entered
from outside, and can it be left?* Those two flags are not authored; they are derived from
the presence of an edge to the reserved enter and exit vertices.

## State

Declared in [`smart_cover_description.h`](smart_cover_description.h.md).

## Construction

**Contract** — takes the authored name of the cover kind. Loads loopholes, loads
transitions, then derives the enter/exit flags. Any failure at any step is fatal and names
the cover kind. Blocks on the script virtual machine throughout.

```text
FUNCTION build_description(table_id)
  load_loopholes(table_id)      # must not be empty; at least one usable
  load_transitions(table_id)
  process_loopholes()           # derive enterable/exitable; at least one of each
```

**Invariants** — transitions must load *after* loopholes because the loopholes are the
graph's real vertices; the graph's own vertex set is built lazily from the transition
endpoints, so a loophole nobody transitions to or from simply never appears in it and is
correctly derived as neither enterable nor exitable.

## `load_loopholes`

**Contract** — resolves the script path
`smart_covers.descriptions.<table_id>.loopholes`, which must be a table, and builds one
loophole per entry. Rejects duplicate loophole names. Fails if the table is missing or
malformed, if there are no loopholes at all, or if none of them is usable.

**Invariants** — the "at least one usable" check is separate from "at least one loophole"
because a loophole becomes unusable by having no actions authored
([`smart_cover_loophole.cpp`](smart_cover_loophole.cpp.md)), which is easy to do by
accident and produces a cover creatures will approach and then stand next to.

**Notes** — the script path is assembled by string concatenation from a fixed prefix. That
prefix is a frozen coupling to the shipped scripts' table layout: the game data declares
smart covers in exactly that namespace, so a rebuild cannot rename it.

## `load_transitions`

**Contract** — resolves `smart_covers.descriptions.<table_id>.transitions` and builds the
graph. Each entry names a source and a destination loophole (empty meaning outside — see
[`smart_cover_detail.cpp`](smart_cover_detail.cpp.md)), a weight, and a list of transition
actions. Vertices are created on first mention. Non-table entries are skipped.

```text
FUNCTION load_transitions(table_id)
  FOR EACH entry IN the transitions table
    IF entry IS NOT a table THEN SKIP
    from   = vertex name of entry.vertex0, read as an *incoming* endpoint
    to     = vertex name of entry.vertex1, read as an *outgoing* endpoint
    weight = REQUIRED entry.weight
    ensure vertices `from` and `to` exist
    add edge from -> to with that weight
    edge.data = one transition action per entry in entry.actions
```

**Invariants** —

- The two endpoints are read with *opposite* direction flags. That is what makes an empty
  source mean "from outside" and an empty destination mean "to outside" in the same table.
- Edges are directed. A cover where a creature can move from one loophole to another but
  not back is legal and authored deliberately — a window you can vault out of but not back
  in through.
- The weight is the plan-search cost, so it is what decides which route through a cover a
  creature takes when several exist.

## `process_loopholes`

**Contract** — for each loophole, sets *enterable* to whether the graph has an edge from
the reserved enter vertex to it, and *exitable* to whether it has an edge to the reserved
exit vertex. Then requires that at least one loophole is enterable and at least one is
exitable.

**Invariants** — this is the only place those two flags are written. They are a property
of the *graph*, not of the loophole, and deriving them rather than authoring them is what
keeps a hand-edited transitions table from disagreeing with a hand-edited loophole table.

## `load_actions`

**Contract** — reads one edge's list of transition actions, building each from its table
via [`smart_cover_transition.cpp`](smart_cover_transition.cpp.md). The list may be empty
in principle; nothing checks.

## `get_loophole`

**Contract** — linear search by name over the loophole list; yields nothing when absent,
rather than failing. This is the one lookup in the smart-cover loader that tolerates a
miss, because callers use it to ask whether a named loophole exists.

**Notes** — the search compares interned string identities rather than characters, which
is what makes a linear scan acceptable for a list this short. A rebuild that compares
strings by content should use a keyed lookup instead.

## Destruction

**Contract** — destroys the loopholes, then walks the graph destroying each vertex's data
and each edge's transition-action list. The graph does not own what its edges point at, so
the walk is explicit.

**Notes** — the explicit walk is entirely an artifact of the original's ownership model.
What survives is the requirement that a description's transition actions and loopholes have
the same lifetime as the description, and that the description outlives every placed cover
that references it — which the reference count in
[`restriction_space.h`](../xrServerEntities/restriction_space.h.md) provides.
