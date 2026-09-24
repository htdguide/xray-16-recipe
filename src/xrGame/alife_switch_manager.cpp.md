# src/xrGame/alife_switch_manager.cpp

> Promotion and demotion: turning an offline record into a live client object and back, without losing state and without leaving an entity registered twice.

**Needs** — [`alife_switch_manager.h`](alife_switch_manager.h.md) · [`xrServer_Objects_ALife.h`](../xrServerEntities/xrServer_Objects_ALife.h.md) · [`alife_graph_registry.h`](alife_graph_registry.h.md) · [`alife_object_registry.h`](alife_object_registry.h.md) · [`alife_schedule_registry.h`](alife_schedule_registry.h.md) · [`xrServer.h`](xrServer.h.md) · [`ai_space.h`](ai_space.h.md) · [`xrAICore/Navigation/level_graph.h`](../xrAICore/Navigation/level_graph.h.md) · [`xrAICore/Navigation/game_level_cross_table.h`](../xrAICore/Navigation/game_level_cross_table.h.md) · [`xrNetServer/NET_Messages.h`](../xrNetServer/NET_Messages.h.md)
**Used by** — reached through its declarations in [`alife_switch_manager.h`](alife_switch_manager.h.md); callers name that, not this file.
**Tier floor** — T2: registry hand-offs and one spawn-message round trip

## Purpose

This is the file conformance criterion 11 is about: *entities cross the boundary into and
out of the detailed simulation without losing state*. An **online** entity is a live
client object — rendered, animated, physically simulated, carrying inventory the client
side owns. An **offline** entity is a record. The transition in both directions must be
lossless, and it must leave the entity registered in exactly one arrangement.

The distance policy also lives here, and it is two radii rather than one.

## State

```text
RECORD SwitchManager
  switch_distance  : real   # the nominal radius, authored
  switch_factor    : real   # the hysteresis fraction, authored
  online_distance  : real   # derived: switch_distance * (1 - switch_factor)
  offline_distance : real   # derived: switch_distance * (1 + switch_factor)
  saved_children   : list<EntityId>   # scratch, used across one demotion
  random           : Random           # its own stream
```

Invariant: `online_distance < switch_distance < offline_distance`, always, because both
are derived from the same pair and the factor is positive. **Promotion uses the smaller
radius and demotion the larger**, so an entity hovering at the boundary does not flip
every frame. That hysteresis band is the single most important number in the offline
simulation's feel, and a rebuild with one radius will thrash visibly.

## `add_online`

**Contract** — materializes a live client object from a server record. The entity must be
on the currently loaded level. Destroys and recreates the underlying server entity through
the spawn message path.

```text
FUNCTION add_online(object, update_registries)
  REQUIRE object's game vertex belongs to the loaded level
  object.online = true

  server.destroy_entity(object)               # remove the record-only instance
  object.flags |= SPAWN_UPDATE                # mark the spawn as carrying update state
  server.process_spawn(new packet, as the server's own client, existing = object)
  object.flags &= ~SPAWN_UPDATE

  object.add_online(update_registries)        # the entity's own promotion behaviour
```

**Invariants** — the entity is **recreated through the spawn message path**, exactly as a
network client would receive it, rather than being handed to the client side directly.
That is the same decision the script spawn path makes (see
[`alife_simulator_script.cpp`](alife_simulator_script.cpp.md)) and for the same reason:
the spawn path is the only code that knows how to build a client object, and going
through it means single player and multiplayer construct live objects identically.

The temporary flag is what tells the spawn path that this spawn carries accumulated update
state rather than being a fresh entity. It is raised and lowered around the single call
because it must not persist onto the record.

A validity check on the entity's navigation vertex is present but disabled in the current
source, with a note that it fired for corpses that ended up outside the navigation map.
That is an honest admission: offline death positions come from authored death points and
should always be valid, and evidently are not in every case. A rebuild should keep the
check and fix the data, or at minimum log rather than ignore.

## `remove_online`

**Contract** — demotes a live client object back to a record. The inverse of the above,
with one extra concern: the entity's children.

```text
FUNCTION remove_online(object, update_registries)
  object.online = false

  saved_children = object.children
  IF object is an inventory owner
    drop from saved_children every child that cannot be saved

  server.destroy(object, reliable and ordered)     # destroys the live object AND its children
  REQUIRE object.children is now empty

  object.id = server.reserve_identifier(object.id) # reclaim the same identifier
  object.add_offline(saved_children, update_registries)
```

**Invariants** — three things happen here that have no counterpart in promotion.

- **The child list is snapshotted before destruction**, because destroying the live object
  destroys its children too and empties the list. The record needs that list to rebuild
  its containment offline.
- **Non-savable children are filtered out of the snapshot**, but only for an inventory
  owner. A creature's transient carried objects — an effect, a thrown grenade in flight —
  are not part of what goes offline with it, and including them would resurrect them on
  the next promotion.
- **The identifier is reclaimed from the allocator.** Destroying the live object returns
  the identifier to the pool; the record must take it straight back, or a later spawn will
  reuse it while the record still claims it. The reservation is expected to return the
  same value.

## `switch_online` / `switch_offline`

**Contract** — thin wrappers that delegate to the entity's own transition and log the
event in a diagnostic build. They exist as the manager's public verbs so that callers do
not reach into the entity, and so that the transition has one instrumented point.

## `synchronize_location`

**Contract** — asked before every switch decision: bring the entity's record into line with
where it actually is. Reports whether the entity should be considered for switching at all.

```text
FUNCTION synchronize_location(object) -> bool
  IF NOT object.used_ai_locations()      -> RETURN true    # not on the navigation map
  IF object has a parent                 -> RETURN true    # carried; its parent decides
  IF object is offline AND its level vertex is invalid -> RETURN true
  RETURN object.synchronize_location()
```

**Invariants** — the three early exits all mean "no synchronization is possible or
needed, carry on", not "skip this entity". Only the fourth case — an entity that is on the
navigation map, is a root, and has a usable vertex — actually reconciles its record, and
only that case can report failure. A failure stops the switch decision for this pass,
which is how an entity whose position cannot currently be resolved is simply left alone.

The diagnostic build additionally verifies that the entity's child list contains no
duplicates, by sorting a copy. That is the "no entity is registered twice" invariant again,
checked at the one place where a duplicate would be silently fatal — because a doubly
listed child is destroyed twice on demotion.

## `try_switch_online`

**Contract** — considers promoting an offline entity.

```text
FUNCTION try_switch_online(object)
  IF object has a parent -> RETURN            # carried items follow their parent
  object.try_switch_online()                  # the entity applies its own distance rule
  IF object is still offline AND NOT object.keep_saved_data_anyway()
    object.clear_client_data()
```

**Invariants** — a carried item is never switched on its own. Its parent's promotion
brings it, and the entity's own child list is how. The diagnostic build asserts the
converse — that a carried item's parent is also offline — and logs an "uncontrolled
situation" when it is not, which is the symptom of a containment link that outlived one of
its ends.

The cleanup on a *failed* promotion is the same idea as the squad's: stale client data
from a previous online period must not survive to be resurrected later. The
`keep_saved_data_anyway` hook is how a few entity kinds opt out — see
[`alife_object.cpp`](alife_object.cpp.md), where the base answer is "no".

## `try_switch_offline`

**Contract** — considers demoting an online entity. Carried items are skipped, with the
mirror-image diagnostic assertion (a carried item's parent must also be online).

## `switch_object`

**Contract** — the per-entity entry point, called by the update manager for every entity
it considers. One decision and two reaping points.

```text
FUNCTION switch_object(object)
  IF object.redundant() -> release(object); RETURN     # reap before doing any work
  IF NOT synchronize_location(object) -> RETURN
  IF object is online -> try_switch_offline(object)
  ELSE                -> try_switch_online(object)
  IF object.redundant() -> release(object)             # reap again: the switch may have emptied it
```

**Invariants** — the redundancy test appears **twice**, and both are necessary. The first
skips an entity that is already dead weight — an empty squad, a consumed container — before
spending any work on it. The second catches an entity that *became* redundant during the
switch, which is exactly what happens when a squad's last member is demoted and the squad
empties.

`release` destroys the entity and its whole containment subtree (see
[`alife_simulator_base.cpp`](alife_simulator_base.cpp.md)), so reaping here is the one
place ordinary gameplay removes an entity from the world.
