# src/xrGame/alife_simulator_base2.cpp

> Registration and deregistration: the exact order in which an entity enters and leaves every alife registry, and what happens when one dies.

**Needs** — [`alife_simulator_base.h`](alife_simulator_base.h.md) · [`alife_object_registry.h`](alife_object_registry.h.md) · [`alife_graph_registry.h`](alife_graph_registry.h.md) · [`alife_schedule_registry.h`](alife_schedule_registry.h.md) · [`alife_story_registry.h`](alife_story_registry.h.md) · [`alife_smart_terrain_registry.h`](alife_smart_terrain_registry.h.md) · [`alife_group_registry.h`](alife_group_registry.h.md) · [`alife_registry_wrappers.h`](alife_registry_wrappers.h.md) · [`relation_registry.h`](relation_registry.h.md) · [`xrServer_Objects_ALife_Items.h`](../xrServerEntities/xrServer_Objects_ALife_Items.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: ordered registry mutation

## Purpose

Three routines, and all three are pure ordering. This is the file that decides what "an
entity exists in the alife simulation" actually means: membership in six registries, a
back-reference to the simulator, a place in its parent's containment list, and its own
registration hook, all established in a fixed sequence and torn down in a different one.

It is split from [`alife_simulator_base.cpp`](alife_simulator_base.cpp.md) for compile
reasons only; read them as one module.

## `register_object`

**Contract** — brings an entity fully into the simulation. Called from every creation
path. Fails hard if the entity is already registered, or if it is an attached item whose
parent already lists it.

```text
FUNCTION register_object(object, add_to_object_registry)
  object.on_before_register()          # 1. the entity's chance to fix up its own fields first

  IF add_to_object_registry
    objects.add(object)                # 2. identity: the entity now resolves by id

  graph.update(object)                 # 3. spatial: placed on the game graph / level
  scheduled.add(object)                # 4. temporal: admitted to the offline rotation
  story_objects.add(object.story_id, object)   # 5. authored addressability
  smart_terrains.add(object)           # 6. job-giving places
  groups.add(object)                   # 7. squads

  setup_simulator(object)              # 8. the entity can now reach the simulation

  IF object is an attached inventory item
    parent = objects.lookup(object.parent_id)
    REQUIRE parent does not already list this item
    parent.children.append(object.id)
    parent.attach(object)              # 9. containment

  IF can_register_objects
    object.on_register()               # 10. the entity's own registration behaviour
```

**Invariants** — every step depends on the ones before it, which is why the order is the
content of the routine:

- `on_before_register` runs **before** any registry sees the entity, because it is where
  an entity declares its position, its vertex validity, and whether it uses navigation
  locations at all (a squad, for instance, disclaims both until it has a member). Every
  registry below reads those fields.
- Identity comes before everything else: steps 3 through 9 may look the entity — or its
  parent — up by identifier.
- The simulator back-reference is installed **after** the registries but **before** the
  entity's own hook, so that the hook may use it.
- Containment is established last of the structural steps, because attaching consults the
  parent, which must itself already be registered. The duplicate check on the parent's
  child list is fatal and unrecoverable: a doubly-listed child is released twice.
- `on_register` is the only step that can be suppressed. The gate exists for bulk loading:
  restoring a saved game registers thousands of entities whose own hooks would each reach
  for a world that is not yet whole, so the hooks are deferred and run in a later pass.
  A rebuild must keep that deferral or find an ordering where every hook is safe at
  registration time — and there is not one, because entity hooks reference each other.

## `unregister_object`

**Contract** — removes an entity from the simulation. Not the inverse order of
registration, and deliberately so.

```text
FUNCTION unregister_object(object, alife_query)
  object.on_unregister()               # 1. the entity's own behaviour, while everything still exists

  IF object is an attached inventory item
    parent = objects.lookup(object.parent_id)
    graph.detach(parent, object, parent.game_vertex, alife_query)   # 2. containment

  objects.remove(object.id)            # 3. identity
  story_objects.remove(object.story_id)
  smart_terrains.remove(object)
  groups.remove(object)

  IF object is offline                 # 5. spatial and temporal, per state
    graph.remove(object, object.game_vertex)
    scheduled.remove(object)
  ELSE IF object has no parent
    graph.level.remove(object, tolerate_missing = NOT object.used_ai_locations())
```

**Invariants** —

- The entity's own hook runs **first**, while every registry is still intact, because that
  is where an entity deregisters itself from things the registries do not know about —
  a creature leaving its smart-terrain job, an item detaching a script binding.
- The final branch is the asymmetry that makes this not a mirror of registration. An
  **offline** entity holds a game-graph slot and a schedule slot, and both must be
  released. An **online** entity holds a level-registry slot instead, and only if it is a
  root — a carried item has no independent presence on the level. Getting this wrong
  leaves a phantom in one registry, which surfaces much later as an entity that cannot be
  found or one that is updated after destruction.
- The tolerance on the level removal keys off whether the entity used navigation
  locations, because an entity that disclaimed them was never added. This is the same
  "re-evaluate the admission predicate on the way out" shape the schedule registry uses.

## `on_death`

**Contract** — the simulation's response to an entity dying, from any cause, online or
offline.

```text
FUNCTION on_death(killed, killer)
  IF killed is a creature
    killed.on_death(killer)            # reputation, statistics, scripted reactions
  IF killed is a squad member AND it belongs to a squad
    groups.lookup(killed.group_id).notify_on_member_death(killed)
```

**Invariants** — the creature's own death handling runs before the squad is told, so that
the squad sees a creature already marked dead. The squad's handler removes the member,
which is what keeps the squad's "every member is alive" assumption true.

Death does **not** destroy the entity here. A corpse is a live server object with zero
health, still registered, still saveable and still lootable; destruction is a separate
decision made much later. That is the difference between this game and one where death is
removal, and it is why the corpse's position matters enough to have authored death points
(see [`alife_simulator_base.cpp`](alife_simulator_base.cpp.md)).
