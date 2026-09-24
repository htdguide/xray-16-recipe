# src/xrGame/alife_dynamic_object.cpp

> The online/offline boundary: what a server record does when it is promoted to a live simulated object, what it does on the way back, and how it decides which side of the boundary it should be on at all.

**Needs** — [`xrServerEntities/xrServer_Objects_ALife.h`](../xrServerEntities/xrServer_Objects_ALife.h.md) · [`alife_simulator.h`](alife_simulator.h.md) · [`alife_schedule_registry.h`](alife_schedule_registry.h.md) · [`alife_graph_registry.h`](alife_graph_registry.h.md) · [`alife_object_registry.h`](alife_object_registry.h.md) · [`Level.h`](Level.h.md) · [`map_manager.h`](map_manager.h.md) · [`xrServer.h`](xrServer.h.md) · [`xrAICore/Navigation/level_graph.h`](../xrAICore/Navigation/level_graph.h.md) · [`xrAICore/Navigation/game_graph.h`](../xrAICore/Navigation/game_graph.h.md) · [`xrAICore/Navigation/game_level_cross_table.h`](../xrAICore/Navigation/game_level_cross_table.h.md) · [Seam: Script virtual machine](../../SYSTEM-REQUIREMENTS.md#seam-script-virtual-machine)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: registry bookkeeping and distance comparisons.

## Purpose

This file implements conformance criterion 11 — entities cross into and out of the detailed
simulation without losing state — for every dynamic server object in the game. It owns the
two transitions, the hysteresis that stops an object oscillating across the boundary, and
the rule for keeping an object's fine position consistent with the two navigation graphs it
is indexed by.

## State

```text
# Carried on every dynamic server object:
  online       : bool                # promoted to a live client object?
  graph_vertex : int                 # its place on the cross-level game graph
  level_vertex : int                 # its place on the loaded level's navigation mesh
  distance     : real                # distance from the level vertex to the game vertex
  position     : point               # its fine world position
  client_data  : bytes               # the last live instance's serialized state,
                                     #   kept across a demotion so the next promotion
                                     #   restores rather than re-creates
  direct_control : bool              # the simulation, not a script, owns its motion

# Invariant: the graph vertex and the level vertex must agree — the cross table maps
#   every level vertex to exactly one game vertex, and the object's graph vertex must
#   be that one. synchronize_location is what restores the agreement after motion.
# Invariant: an object registered on the graph registry is offline; an online object is
#   registered with the live level instead. Being in both is the bug the registries'
#   duplicate assertions exist to catch.
```

## `switch_online` / `switch_offline`

**Contract** — Flip the online flag and tell the simulation. Each asserts the object was in
the *other* state first, so a double promotion or a double demotion is a hard error rather
than a silently corrupted registry. Demotion additionally clears the saved client state.

**Invariants** — The flag is set *before* the simulation is told, so any registry the
simulation touches during the call already sees the new state.

## `add_online` / `add_offline`

**Contract** — The registry half of each transition, skipped entirely when the caller says
it will do the bookkeeping itself.

```text
FUNCTION add_online()      # becoming live
  remove from the coarse scheduler        # the live object gets frame updates instead
  remove from the graph registry          # it is no longer a graph-vertex occupant

FUNCTION add_offline()     # becoming a record again
  add to the coarse scheduler
  add to the graph registry at its graph vertex
```

**Invariants** — The two are exact inverses, and the pair is the whole meaning of *online*:
an online object is driven by the frame loop and indexed by the level; an offline one is
driven by the coarse scheduler and indexed by the cross-level graph. Nothing is ever in
both.

## `try_switch_online`

**Contract** — Asks whether an offline object should be promoted, and promotes it if so.
Also keeps the object's coarse scheduler registration in step with whether it still needs
coarse updates at all — a creature that died offline stops being scheduled.

```text
FUNCTION try_switch_online()
  IF this object is schedulable
    IF it no longer needs updates THEN ensure it is not scheduled
    ELSE ensure it is scheduled

  IF it cannot go online          THEN on_failed_switch_online(); RETURN
  IF it cannot go offline         THEN switch_online(); RETURN   # it must be live
  IF distance(actor, this) > online_distance
                                  THEN on_failed_switch_online(); RETURN
  switch_online()
```

## `try_switch_offline`

**Contract** — The mirror. An object that cannot be offline is never demoted; an object
that cannot be online is demoted unconditionally; otherwise the player's distance decides.

```text
FUNCTION try_switch_offline()
  IF it cannot go offline         THEN RETURN
  IF it cannot go online          THEN switch_offline(); RETURN   # it must be a record
  IF distance(actor, this) <= offline_distance THEN RETURN
  switch_offline()
```

**Invariants** — The promotion threshold and the demotion threshold are *different
distances*, and the demotion one is larger. That gap is the hysteresis band: without it an
object sitting exactly at the threshold would be promoted and demoted on alternating
frames, which is expensive and visible. Any rebuild must keep two thresholds.

Both routines are written so that an object which can only exist in one state is forced
into it before distance is ever consulted. The player's own record, for instance, can never
go offline.

## `synchronize_location`

**Contract** — Reconciles an object's fine position with the two graphs after it has moved.
Returns unconditionally true — the result is vestigial. Does nothing when the position is
outside the level graph entirely, or when it is still inside the level vertex the object
already claims.

```text
FUNCTION synchronize_location()
  IF the position is not a valid level-graph position THEN RETURN
  IF the position is still inside the claimed level vertex THEN RETURN

  new_vertex = the level vertex nearest the position, searched from the claimed one
  IF offline AND the position is not inside new_vertex THEN RETURN
      # An offline object is only moved when the answer is exact. Online objects
      # accept the nearest vertex, because a live object standing on a ledge or in
      # a doorway must still be indexed somewhere.

  level_vertex = new_vertex
  new_graph_vertex = cross_table[level_vertex].game_vertex

  IF new_graph_vertex differs from the current one
    IF offline
      # Moving between graph vertices resets the fine placement, so save it and
      # put it back when the new vertex still contains it.
      save (position, level_vertex)
      graph registry.change(this, old graph vertex, new graph vertex)
      IF the saved position is inside the saved level vertex
        restore both
    ELSE
      REQUIRE the new graph vertex belongs to the loaded level
      graph vertex = new_graph_vertex      # no registry move: online objects are
                                           #   not in the graph registry
  distance = cross_table[level_vertex].distance
```

**Invariants** — The asymmetry between online and offline is the point of the routine. An
*online* object's graph vertex is a label — it is indexed by the level, so the field is
updated directly and asserted to stay within the loaded level. An *offline* object's graph
vertex is its index, so changing it is a registry move with all the bookkeeping that
implies.

The save-and-restore around the graph move is the same dance as in
[`alife_anomalous_zone.cpp`](alife_anomalous_zone.cpp.md), for the same reason: a graph
move is modelled as a teleport to the vertex's own level point, and a caller who computed a
better position must defend it.

## `on_register` and the client-state rule

**Contract** — When a record enters the object registry, it walks up its parent chain to the
outermost container and asks whether *that* object is present on the loaded level. If it is
not, the record's saved client state is discarded.

**Invariants** — Saved client state is the serialized form of a live instance, and it is
only meaningful for the level it was recorded on. A record carried in somebody's backpack
across a level boundary must not be restored from a snapshot taken elsewhere. Walking to
the outermost parent is what makes the test apply to the *container*, since an item inside
a box inside a truck follows the truck.

## `on_unregister`

**Contract** — Two notifications on the way out: the script layer is given a chance to react,
by calling a well-known global function with the entity identifier if the script side
defines one, and the map's marker manager is told to drop any marker for this entity.

**Notes** — The script hook is discovered by name at call time rather than registered, which
means a mod can add the hook without the engine knowing. The name is frozen by that
contract.

## `clear_client_data` / `on_failed_switch_online`

**Contract** — Discard the saved client state, unless the object has declared that its saved
state must be kept regardless (anomalies do; see
[`alife_anomalous_zone.cpp`](alife_anomalous_zone.cpp.md)). A *failed* promotion clears it
too, which is the non-obvious part: an object that tried to go online and could not has
state that may no longer be restorable, so it is dropped rather than risked.

## Inventory box promotion and demotion

An inventory box is a container whose contents must exist as real objects while it is live
and must collapse back into records when it is not. It overrides both transitions.

**Contract (going online)** — For every child, mark it as needing a spawn update, destroy
its existing server-side entity, move it to the box's own position and level vertex, and
re-spawn it through the server's spawn path as an online object. Then run the ordinary
promotion.

**Invariants** — Children are placed at the *box's* position, not at their own remembered
one: an item inside a box has no independent position, and restoring its old one would
scatter the box's contents across the level.

**Contract (going offline)** — For every child that was saved: clear its online flag,
**re-issue its entity identifier** through the server's generator, drop it entirely if it
cannot be saved, clear its client state, move it through the graph registry, and record it
as a child of the box.

**Invariants** — The identifier re-issue is the load-bearing step and the reason this
override exists. Entity identifiers are 16 bits and are recycled; an item that existed as a
live object held one from the live pool, and as a stored record it needs one from the
persistent pool. Skipping the re-issue produces two entities with one identifier the next
time the box is opened.

A child that cannot be saved is released rather than stored, and the loop adjusts its own
bounds to account for the removal — the shipped code does this by decrementing both the
index and the count, which is correct only because the released child is removed from the
list being walked. A rebuild should iterate over a snapshot.

## `redundant`

**Contract** — Always false for the base dynamic object: nothing is redundant by default.
Subclasses override to let the simulation cull records it no longer needs.
