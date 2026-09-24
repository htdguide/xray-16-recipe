# src/xrGame/alife_simulator_base.cpp

> Owns every alife registry, and owns entity creation: how a spawn record, a configuration section or a group template becomes a live server object.

**Needs** — [`alife_simulator_base.h`](alife_simulator_base.h.md) · [`alife_simulator_header.h`](alife_simulator_header.h.md) · [`alife_time_manager.h`](alife_time_manager.h.md) · [`alife_spawn_registry.h`](alife_spawn_registry.h.md) · [`alife_object_registry.h`](alife_object_registry.h.md) · [`alife_graph_registry.h`](alife_graph_registry.h.md) · [`alife_schedule_registry.h`](alife_schedule_registry.h.md) · [`alife_story_registry.h`](alife_story_registry.h.md) · [`alife_smart_terrain_registry.h`](alife_smart_terrain_registry.h.md) · [`alife_group_registry.h`](alife_group_registry.h.md) · [`alife_registry_container.h`](alife_registry_container.h.md) · [`inventory_upgrade_manager.h`](inventory_upgrade_manager.h.md) · [`object_factory.h`](../xrServerEntities/object_factory.h.md) · [`xrServer.h`](xrServer.h.md) · [`Level.h`](Level.h.md) · [`xrAICore/Navigation/level_graph.h`](../xrAICore/Navigation/level_graph.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: registry ownership, a factory and a packet round trip

## Purpose

This layer is where the alife simulation's *contents* live. It constructs and destroys the
eleven sub-objects the simulation is made of, and it holds the several spellings of
"create an entity" that the rest of the game calls. Registration and death handling live
in the sibling [`alife_simulator_base2.cpp`](alife_simulator_base2.cpp.md); the accessors
are in
[`alife_simulator_base_inline.h`](alife_simulator_base_inline.h.md).

The split across three files is arbitrary — it was a compile-time measure — and a rebuild
should read the three as one module.

## State

```text
RECORD ALifeSimulatorBase
  server               : ref NetworkServer     # not owned; supplies identifier allocation
  header               : SimulatorHeader       # world identity and save version
  time_manager         : TimeManager           # game clock
  spawns               : SpawnRegistry         # the level's authored spawn records
  objects              : ObjectRegistry        # every live server object
  graph_objects        : GraphRegistry         # entities indexed by game-graph vertex, plus the level registry
  scheduled            : ScheduleRegistry      # the offline update rotation
  story_objects        : StoryRegistry         # entities addressable by authored story identifier
  smart_terrains       : SmartTerrainRegistry
  groups               : GroupRegistry         # squads
  registry_container   : RegistryContainer     # the persistent per-character stores
  upgrade_manager      : InventoryUpgradeManager
  random               : Random32              # the simulation's own stream
  initialized          : bool
  server_command_line  : ref text
  can_register_objects : bool
```

Invariants:

- every sub-object exists exactly while `initialized` holds, and **every accessor asserts
  that**, so reaching into the simulation before it is built or after it is torn down is a
  diagnosed fault rather than a crash;
- the random stream belongs to the simulation, not to the process. It is seeded from the
  processor's cycle counter at construction, so a session is *not* reproducible across
  runs. That is worth flagging against conformance criterion 8, which asks for
  determinism: the physics is deterministic given a seed, and this seed is not recorded.
  A rebuild that wants reproducible sessions must persist it;
- `can_register_objects` gates the entity's own registration hook and nothing else; see
  the note under registration.

## Construction and `reload`

**Contract** — construction only zeroes the sub-object slots, seeds the random stream and
records the server. Nothing is built. `reload` is what builds them, from a named
configuration section, and raises `initialized`.

```text
FUNCTION reload(section)
  header             = new SimulatorHeader(section)
  time_manager       = new TimeManager(section)
  spawns             = new SpawnRegistry(section)
  objects            = new ObjectRegistry(section)
  graph_objects      = new GraphRegistry()
  scheduled          = new ScheduleRegistry()
  story_objects      = new StoryRegistry()
  smart_terrains     = new SmartTerrainRegistry()
  groups             = new GroupRegistry()
  registry_container = new RegistryContainer()
  upgrade_manager    = new InventoryUpgradeManager()
  initialized        = true
```

**Invariants** — construction and building are separate because the session's parameters
are not known until after the base exists (the derived simulator's constructor validates
them). Only four of the eleven take a configuration section; the rest have nothing to
tune. A rebuild should pass the section only where it is read, and should note that
`initialized` is raised at the *end*, so a failure partway leaves the simulation
unusable rather than half-usable.

## `unload`

**Contract** — destroys every sub-object and lowers `initialized`, then tells the loaded
level that the simulation is gone.

**Invariants** — the object registry is destroyed **first**, before every other registry.
That order is deliberate and load-bearing: destroying it runs every entity's unregister
hook, and those hooks reach into the graph, story, smart-terrain and group registries to
deregister themselves. Freeing any of those first would hand a live hook a dead registry.

The level notification comes last, after everything is gone, because the level's own
teardown assumes the simulation no longer exists.

## `spawn_item` — create an entity from a configuration section

**Contract** — the general "make me one of these" entry point, used by scripts, by loadout
generation and by anything that spawns an item at a position. Allocates an entity
identifier from the server. Fails hard if the section names no known entity class.
Returns the new server object, already registered unless the caller asked otherwise.

```text
FUNCTION spawn_item(section, position, level_vertex, game_vertex, parent_id, register?) -> ServerObject
  entity = entity_factory.create(section)     # section name doubles as the class key
  REQUIRE entity exists  ELSE FAIL WITH "cannot find item with section"

  entity.section     = section
  entity.respawn_point = none
  entity.id          = server.allocate_identifier()
  entity.parent_id   = parent_id
  entity.phantom_id  = none
  entity.position    = position
  entity.spawn_version = SPAWN_VERSION
  entity.display_name  = section + zero_padded(entity.id, width 4)

  IF entity is a weapon
    entity.rounds_in_magazine = its magazine size      # weapons arrive loaded

  entity.level_vertex = level_vertex
  entity.game_vertex  = game_vertex
  entity.spawn_record = none                  # not from the level's spawn file

  IF register?
    register_object(entity, add_to_object_registry = true)

  entity.spawn_supplies()                     # its own embedded loadout, if any
  entity.on_spawn()
  RETURN entity
```

**Invariants** —

- **Registration precedes the spawn hooks.** An entity's loadout spawns children that
  reference it as their parent, and its spawn hook may look itself up; both require the
  entity to already be in the object registry. Reversing these produces children whose
  parent identifier resolves to nothing.
- The display name is the section name followed by the identifier zero-padded to four
  digits. That is not cosmetic: it is the key scripts and the debugger address entities
  by, and it appears in log output and in save diagnostics, so its exact shape is part of
  the modding surface.
- A spawned entity carries no spawn-record identifier. That is how the save and the switch
  machinery tell a scripted spawn from an authored one — an authored entity can be
  recreated from the level's spawn file, a scripted one can only be restored from the
  save.
- A weapon arrives with a full magazine. This is the one class-specific special case in a
  generic factory, and it exists because the shipped loadout data assumes it.

## `create(out, template, spawn_id)` — create from the level's spawn file

**Contract** — instantiates the entity an authored spawn record describes, by copying the
record's state through a serialized round trip, and then does the extra work an authored
entity needs. Fails hard on an unknown class or a non-alife class.

```text
FUNCTION create(OUT entity, template, spawn_id)
  entity = entity_factory.create(template.section)
  REQUIRE it exists and is an alife object

  # Clone the template's state via the wire format rather than by copying fields.
  packet = template.write_spawn(as_save = true)
  entity.read_spawn(packet)
  packet = template.write_update()
  entity.read_update(packet)

  REQUIRE entity does not use navigation locations OR its level vertex is valid

  entity.spawn_record = spawn_id

  # The player is entity zero, always, and only if no player exists yet.
  IF entity is the player record AND no player is registered
    entity.id = 0
  ELSE
    entity.id = server.allocate_identifier()

  register_object(entity, add_to_object_registry = true)
  entity.controlled_by_alife = true

  IF entity is a creature
    graph.assign(entity)                # place it on the game graph

  IF entity is a group template
    members = list of size entity.member_count
    FOR EACH slot IN members
      slot = create_group_member(entity, template).id
  ELSE
    entity.spawn_supplies()

  entity.on_spawn()
```

**Invariants** —

- **The player is entity identifier zero.** Every other identifier is allocated. This is
  relied on throughout the game and the network protocol, and the guard ("only if no
  player is registered yet") is what stops a second player record in a level that
  mistakenly contains one.
- Cloning through the **serialized form** rather than by field copy is the decision worth
  preserving. The spawn file's records are read as templates, and a template may describe
  a class whose fields this code knows nothing about; round-tripping through the same
  bytes the network and the save use guarantees the copy is complete for any class,
  present or future. The cost is two buffer copies per entity at level load.
- A group template does **not** spawn supplies; its members do. A group is a count and a
  member section, not an entity with pockets.
- The navigation-vertex check is a precondition, not a repair: an authored entity that
  claims to use navigation locations but has no valid vertex is a broken level, and the
  engine says so rather than guessing a vertex.

## `create_group_member(group, template)` — one creature of a group

**Contract** — instantiates one member of an authored group. The member's class comes from
the group's configuration section, not from the template. Fails hard on an unknown class.

```text
FUNCTION create_group_member(group, template) -> ServerObject
  member_section = configuration(group.section, "monster_section")
  member = entity_factory.create(member_section)
  REQUIRE it exists and is an alife object

  copy template's spawn and update state into member    # same round trip as above
  member.section       = member_section
  member.spawn_record  = template.spawn_record
  member.id            = server.allocate_identifier()
  member.direct_control = false
  member.controlled_by_alife = true
  member.display_name  = member_section + zero_padded(member.id, width 4)

  register_object(member, add_to_object_registry = true)
  member.spawn_supplies()
  member.on_spawn()
  RETURN member
```

**Invariants** — every member is cloned from the *same* template and therefore starts at
the same position with the same state; they are separated afterwards by the graph
assignment and by their own movement. The members share the group's spawn-record
identifier, which is what lets the whole group be recognised as coming from one authored
record.

`direct_control` is cleared and alife control set: a group member is never a
player-controlled or network-controlled entity.

## `create(object)` — adopt an entity the client created

**Contract** — the path by which an object spawned on the client side (a dropped item, a
fired grenade, anything the live simulation produced) becomes an alife server object.

```text
FUNCTION create(object)
  IF object is not a dynamic alife object -> RETURN
  IF NOT object.can_save()
    object.controlled_by_alife = false      # transient: the simulation will not track it
    RETURN
  REQUIRE object is online

  IF object has a parent
    parent = objects.lookup(object.parent_id)
    object.game_vertex, position, level_vertex = parent's
    # Register it as a root, then restore the parent link.
    saved_parent   = object.parent_id
    object.parent_id = none
    register_object(object, add_to_object_registry = true)
    object.parent_id = saved_parent
  ELSE
    register_object(object, add_to_object_registry = true)
```

**Invariants** — the parent link is temporarily cleared across registration. That looks
like a hack and is one, but the reason is structural: registration's attachment step would
otherwise try to attach the item to its parent a second time, and the item is *already*
attached on the client side. Clearing the link makes registration treat it as a root; the
link is restored immediately so the containment tree is correct. A rebuild with an
explicit "register without attaching" flag expresses the same thing honestly.

The position is inherited from the parent before registration, because a carried item has
no position of its own and the registries need one.

An object that cannot be saved is registered nowhere and merely disclaims alife control.
That is how bullets and effects exist on the client without ever entering the simulation.

## `release` — destroy an entity and everything it contains

**Contract** — removes an entity and its whole containment subtree from the simulation,
optionally asking the server to destroy the underlying objects too. Fails hard if the
entity is not registered.

```text
FUNCTION release(entity, destroy_on_server)
  object = objects.lookup(entity.id)        # REQUIRE present
  IF object has children
    children = a snapshot of object.children      # snapshot BEFORE recursing
    FOR EACH child_id IN children
      child = objects.lookup(child_id, tolerate_missing = true)
      IF child is absent -> CONTINUE
      release(child, destroy_on_server)
  unregister_object(object, destroy_on_server)
  object.controlled_by_alife = false
  IF destroy_on_server
    server.destroy_entity(entity)
```

**Invariants** — **the child list is snapshotted before the recursion**, because
releasing a child mutates the parent's child list. Iterating the live list while it is
being edited is the bug this copy exists to prevent, and it is the single thing a rebuild
must not optimize away.

Children are released **before** the parent, depth first, so that a child's
deregistration can still resolve its parent.

A child identifier that no longer resolves is skipped rather than treated as corruption —
the same tolerance the save walk uses, and for the same reason.

## `assign_death_position` — where a creature's body ends up

**Contract** — kills a creature record and places it somewhere plausible on the game
graph. Two cases.

```text
FUNCTION assign_death_position(creature, graph_vertex, killer?)
  creature.health = 0

  IF killer is an anomalous zone
    spawns.assign_artefact_position(killer, creature)   # the zone may produce an artefact
    IF creature is a creature with graph travel state
      creature.previous_vertex = creature.next_vertex = creature.game_vertex
    RETURN

  # Otherwise: pick one of the vertex's authored death points at random.
  death_points = game_graph.death_points_of(graph_vertex)
  chosen = death_points[random_below(death_points.count)]   # or the first when there are none
  creature.game_vertex  = graph_vertex
  creature.position     = chosen.level_point
  creature.level_vertex = chosen.level_vertex
  creature.distance     = chosen.distance
  REQUIRE the vertex is on another level OR the level vertex is valid
  IF creature has graph travel state
    creature.previous_vertex = creature.next_vertex = creature.game_vertex
```

**Invariants** — **death points are authored per game-graph vertex** and shipped with the
level. A creature that dies offline does not die where it was: it dies at one of the
places the level designer said bodies may be found. That is what keeps corpses out of
walls and off the navigation mesh's edges, and it is the reason offline deaths are
survivable at all as a mechanic.

Clearing the travel state (previous and next graph vertex both set to the current one) is
necessary because a corpse must not continue a journey; leaving it would have the body
interpolating toward a destination.

The anomaly case short-circuits entirely: a creature killed by an anomalous zone stays
where the zone caught it, and the zone gets the chance to convert the death into an
artefact. That is the in-fiction explanation of where artefacts come from, implemented as
a special case in the death path.

The random draw uses the *simulation's* stream, so the choice is part of the simulation's
sequence rather than of the renderer's or the client's.

## `append_item_vector`

**Contract** — given a list of entity identifiers, appends those that are inventory items
to a list of item references, skipping the rest. A small convenience over the object
registry, used when a container's contents must be examined as items rather than as
entities.

## `level_name`

**Contract** — the authored name of the level currently loaded, taken from the game graph's
header via the level graph's identifier. Requires a loaded level.
