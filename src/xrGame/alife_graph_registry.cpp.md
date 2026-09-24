# src/xrGame/alife_graph_registry.cpp

> The index of the whole offline world by cross-level graph vertex, the trigger that loads a level when the player's record first appears, and the single place where an item changing hands is reflected in both the owner's list and the world index.

**Needs** — [`alife_graph_registry.h`](alife_graph_registry.h.md) · [`alife_graph_registry_inline.h`](alife_graph_registry_inline.h.md) · [`alife_level_registry.h`](alife_level_registry.h.md) · [`xrServerEntities/xrServer_Objects_ALife_All.h`](../xrServerEntities/xrServer_Objects_ALife_All.h.md) · [`xrServerEntities/xrMessages.h`](../xrServerEntities/xrMessages.h.md) · [`ai_space.h`](ai_space.h.md)
**Used by** — [`alife_graph_registry.h`](alife_graph_registry.h.md)
**Tier floor** — T3: an array of keyed tables plus one load trigger.

## Purpose

The off-screen simulation needs to answer "what is at this place in the world" cheaply and
constantly. This file is that index: one table of objects per **game graph** vertex, plus a
secondary index from terrain classification to the vertices carrying it, plus the subset of
the index belonging to the currently loaded **level**.

It also owns a decision that looks out of place and is not: the moment the player's record
is registered is the moment the engine knows which level to load, so the level load is
triggered from here.

## State

```text
RECORD GraphRegistry
  objects  : list<map<entity id, object>>     # one table per game graph vertex,
                                              #   indexed by vertex identifier
  terrain  : list of vertex lists, indexed by [classification axis][classification value]
  level    : optional level registry          # the subset on the loaded level
  actor    : optional reference to the player's record
  process_time : real                         # the simulation's per-frame time budget,
                                              #   forwarded to the level registry
  pending  : list<object>                     # objects added before a level existed

# Invariant: objects[v] holds exactly the offline objects whose graph vertex is v and
#   which use navigation locations. Online objects and location-less ones are absent.
# Invariant: the level registry exists only once the player's record has been seen; the
#   pending list exists to hold objects registered before that moment.
# Invariant: each object appears in at most one vertex table. Moving between vertices
#   is remove-then-add, never a copy.
```

## `on_load`

**Contract** — Sizes the index to the game graph and builds the terrain index. Clears
everything. Called once per game graph load.

```text
FUNCTION on_load()
  FOR EACH classification axis
    clear its buckets
    FOR EACH game graph vertex
      append the vertex to the bucket for its value on that axis
  size the per-vertex object tables to the vertex count, all empty
```

**Notes** — The terrain index is a *transpose* of data the game graph already holds: each
vertex carries a small fixed set of classification values (the shipped graphs use four
axes) and this inverts the mapping so the simulation can ask "give me every vertex of this
kind" without scanning. It is built once and never updated, because the game graph is
immutable.

## `update` — registering an object, and the level-load trigger

**Contract** — Called as each object joins the simulation. Ignores objects the simulation
does not directly control. Recognizes the player's record, applies a pending start-position
override if one is set, and on first sight of the player loads the level. Finally indexes
the object, unless it is an item currently held by somebody.

```text
FUNCTION update(object)
  IF NOT object.direct_control THEN RETURN

  IF object is flagged as the player
    REQUIRE it really is an actor record
    actor = object
    IF a start game vertex was requested
      move the actor to it and to the requested start position

  IF actor is known AND no level is loaded yet
    setup_current_level()

  IF the object is an attached inventory item THEN RETURN   # its owner holds it
  add(object, object's graph vertex)
```

**Invariants** — An attached item is deliberately *not* indexed by place. Its place is its
owner's, and indexing it separately would let the simulation find a rifle lying on a graph
vertex while somebody is carrying it. The whole attach/detach pair below exists to maintain
that.

The start-position override is a one-shot: it is consumed and cleared by the level load, so
starting a new game places the player where the game wants and loading a save does not.

## `setup_current_level`

**Contract** — Creates the level-scoped subset for the level the player's record is on,
fills it from every graph vertex belonging to that level plus anything that arrived early,
and then actually loads the level's data. Fails hard if the level named by the graph does
not exist on disk.

```text
FUNCTION setup_current_level()
  level = new level registry for the actor's graph vertex's level
  hand it the current time budget
  FOR EACH game graph vertex on that level
    FOR EACH object indexed at that vertex
      add it to the level registry
  FOR EACH object in the pending list
    add it to the level registry
  clear the pending list
  look the level's name up in the graph's level table; FAIL if absent
  REQUIRE the level exists in the game data
  ai space.load(level name)          # graphs, cover, doors — see ai_space.cpp
  clear the start-vertex override
```

**Invariants** — The pending list is drained here and only here. Objects registered before
the player's record was seen have nowhere to go — the level subset does not exist yet — so
they are parked and adopted at this moment. A rebuild that creates the level subset up
front deletes the pending list entirely.

## `add` / `remove`

**Contract** — Put an object into, or take it out of, the index at a given graph vertex, and
mirror the change into the level subset. Three distinct cases, selected by the object's
state.

```text
FUNCTION add(object, vertex, also_update_level = true)
  IF object is offline AND uses navigation locations
    REQUIRE vertex is a valid graph vertex
    insert into objects[vertex]; object.graph_vertex = vertex
  ELSE IF no level exists yet AND also_update_level
    park it in the pending list; object.graph_vertex = vertex
  IF also_update_level AND a level exists AND vertex is valid
    level.add(object)

FUNCTION remove(object, vertex, also_update_level = true)
  IF object uses navigation locations
    erase from objects[vertex]
  IF also_update_level AND a level exists
    level.remove(object, leaving = the vertex's level is not the loaded one)
```

**Invariants** — Objects that do not use navigation locations — abstract records with no
place in the world — are never in the vertex tables but *are* in the level subset. That
split is why the two tests differ between the two halves of each function, and it is easy
to get wrong: adding is conditional on being offline, removing is not, because an object
may be removed as part of going online and by then the flag has already flipped.

The removal tells the level subset *whether the object is leaving the level* or merely
moving within it, which the subset uses to decide whether the removal is permanent.

## `attach` / `detach`

**Contract** — The two halves of an item changing hands, each doing the world-index side and
the ownership side in one call so they cannot drift apart.

```text
FUNCTION attach(owner, item, vertex, through_the_simulation = true, cascade = true)
  IF through_the_simulation
    remove(item, vertex)         # it is no longer lying at a place...
  ELSE
    level.remove(item)           # ...or at least not on this level
  REQUIRE the owner is a simulation object when going through the simulation
  owner.attach(item, cascade)    # ...it is now somebody's

FUNCTION detach(owner, item, vertex, through_the_simulation = true, cascade = true)
  IF through_the_simulation
    add(item, vertex)            # it is lying at a place again
  ELSE
    item.graph_vertex = vertex; level.add(item)
  owner.detach(item, cascade)
```

**Invariants** — The world-index change happens *before* the ownership change in both
directions. That ordering means there is never an instant where the item is both indexed at
a place and listed as a child; the transient inconsistency is always the harmless one,
where it is briefly neither.

The cascade flag controls whether the item's own children follow it. Attaching a backpack
attaches what is in it.

**Notes** — Detaching from a non-simulation owner is a case the code notices and does not
handle: it logs an error in checked builds when the item is not actually the owner's child
and the assertion that would have caught it is commented out. That is an accepted, known
looseness — some items are owned by things the simulation does not track — and a rebuild
should decide whether such owners exist rather than inherit the ambiguity.

## Construction and destruction

**Contract** — Starts with no level subset, no player record and a zero time budget.
Destroying the registry destroys the level subset; the per-vertex tables do not own the
objects, which belong to the object registry.
