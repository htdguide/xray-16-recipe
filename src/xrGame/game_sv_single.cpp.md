# src/xrGame/game_sv_single.cpp

> The single-player session: a game mode whose entire rule set is "there is an alife simulation, and it decides".

**Needs** — [`game_sv_single.h`](game_sv_single.h.md) · [`alife_simulator.h`](alife_simulator.h.md) · [`alife_object_registry.h`](alife_object_registry.h.md) · [`alife_graph_registry.h`](alife_graph_registry.h.md) · [`alife_time_manager.h`](alife_time_manager.h.md) · [`xrServer.h`](xrServer.h.md) · [`xrServer_Objects_ALife_Monsters.h`](../xrServerEntities/xrServer_Objects_ALife_Monsters.h.md) · [`GamePersistent.h`](GamePersistent.h.md)
**Used by** — reached through its declarations in [`game_sv_single.h`](game_sv_single.h.md); callers name that, not this file.
**Tier floor** — T2: session policy over the alife simulation; no device contact

## Purpose

Every game mode answers the same questions — what time is it, may this entity pick that one
up, what happens when someone dies, what does a save contain. In single player every one of
those answers belongs to the alife simulation, so this file is almost entirely a forwarding
layer with one important twist: **the simulator is optional**, and every method must work
without it.

That optionality is not defensive coding. The engine can run a single-player level with the
alife simulation switched off — for the editor, for a standalone test map, for a level
loaded outside a campaign — and in that mode the session must degrade to the base game
state's answers rather than fail. The pattern "if there is a simulator ask it, otherwise ask
the base" repeats in every time and persistence method and is the file's shape.

## State

```text
RECORD SingleSession
  alife : optional<AlifeSimulator>   # absent when the session was created without the
                                     # "/alife" option; every method must cope
  type  : GameType = single
```

**Invariants** — the simulator is created during session creation and destroyed with the
session. `restart_simulator` is the single exception that replaces it mid-session, and it
must put the server's identifier allocator back to a clean state first.

## `Create`

**Contract** — runs the base session creation, then creates an alife simulator **only if the
session options string contains the alife switch**, then moves the session straight to the
in-progress phase. Single player has no pending or warm-up phase: the game is running the
moment it is created.

```text
FUNCTION create(options : text)
  base.create(options)
  IF options contains "/alife" THEN
    alife = new AlifeSimulator(server, options)   # the simulator parses the rest of
                                                  # the option string itself
  switch_phase(in_progress)
```

**Notes** — the option string is passed to the simulator *by reference* and the simulator
keeps it: `restart_simulator` later reads it back out of the simulator to rebuild with the
same options. A rebuild must make the session's option string outlive the simulator that
consumed it.

## `OnCreate`

**Contract** — called when a server object is created. Decides whether the new object joins
the alife simulation's own registry, and if not, marks it as not alife-controlled so nothing
later tries to advance it offline.

```text
FUNCTION on_create(entity_id)
  IF no alife simulation THEN RETURN
  entity = server_object(entity_id)
  IF NOT entity.alife_controlled THEN RETURN      # an entity the level spawned directly
  IF entity is not an alife object THEN RETURN

  entity.online = true

  IF entity.parent_id is none THEN
    alife.create(entity)                          # a free-standing world object
  ELSE
    parent = alife.registry.object(entity.parent_id)
    IF parent exists AND (parent is a trader OR parent is an inventory box) THEN
      alife.create(entity)                        # contents of a container the alife
                                                  # simulation owns are owned by it too
    ELSE
      entity.alife_controlled = false             # carried by a creature, or the parent
                                                  # is not registered: the owner is
                                                  # responsible for it, not the simulation
```

**Invariants** — the rule is *who owns this object's persistence*. Only two kinds of parent
delegate that back to the simulation: a trader and an inventory box, both of which are
stores whose contents must survive independently of any live object. Anything else carrying
an item — a creature, the player — serializes its own inventory, so registering the item
separately would save it twice and restore it in two places.

A missing parent is treated the same as a non-store parent: not alife-controlled. Silently,
because a missing parent at this moment is normal during a level load where the parent has
not been created yet.

## `OnTouch`

**Contract** — an entity has taken ownership of another. If both are registered with the
alife simulation and the item is currently present on the loaded level's graph, the
ownership move is recorded in the alife graph so that the item travels with its new owner
when either goes offline. Always permits the transfer — single player never refuses a
pickup by rule.

**Invariants** — three conditions must all hold before the graph is told: the item is an
alife inventory item, the taker is an alife dynamic object, and the item is registered on
the level's graph *and* both are in the object registry. Recording an attachment for an
unregistered object corrupts the graph's ownership tree, which is the structure a save is
written from.

**Notes** — the graph attachment is told the taker's *graph vertex*, not its position: alife
ownership is recorded against the coarse cross-level graph because that is the only position
an offline object has.

## `OnDetach`

**Contract** — an entity has given up ownership. Two cases, and the second is the
interesting one.

```text
FUNCTION on_detach(owner_id, item_id)
  IF no alife simulation THEN RETURN
  IF item is not an alife inventory item THEN RETURN
  IF owner is not an alife dynamic object THEN RETURN

  IF owner is registered AND item is registered
     AND item is NOT on the loaded level's graph THEN
    alife.graph.detach(owner, item, owner.graph_vertex)
  ELSE IF item is NOT in the object registry THEN
    # the item never existed to the simulation: the owner carried it as its own state.
    # Dropping it promotes it to a world object of its own, at the owner's place.
    remembered_parent = item.parent_id
    item.parent_id    = none            # create() refuses an object that claims a parent
    item.level_vertex = owner.level_vertex
    item.graph_vertex = owner.graph_vertex
    item.alife_controlled = true
    item.online           = true
    alife.create(item)
    item.parent_id    = remembered_parent   # the caller still needs the old link
```

**Invariants** — the temporary clearing and restoring of the item's parent identifier is the
load-bearing trick: registration refuses a parented object, but the caller's own bookkeeping
still needs the parent link after the call returns. A rebuild whose registration takes the
parent as an argument rather than reading it off the object avoids the dance entirely; what
must be preserved is that the item ends up registered, positioned at the dropper, online and
alife-controlled, *and* the caller's parent link is unchanged.

## Time

**Contract** — four clocks, each of which asks the simulator if it exists and is initialized,
and otherwise falls back to the base session.

- **start game time** — the in-world timestamp the campaign began at.
- **game time** and **time factor** — the current in-world clock and how fast it runs.
- **set time factor**, in two forms: change the rate from now, or schedule a rate change at
  a given in-world time.
- **environment time** and **environment time factor** — a *second* clock, used by the
  weather and time-of-day system.

**Invariants** — the environment clock reads the simulator's time but writes and reads its
*factor* through the base session only. That is the whole reason two clocks exist: a script
can accelerate the world's clock for a timed event (sleeping, waiting) without the sky
slewing at the same rate, and can do the reverse for a cut-scene. A rebuild that collapses
them into one clock will make sleep look like time-lapse.

**Notes** — "initialized" is tested separately from "exists" because the simulator is
constructed before its time manager has read the save; asking it the time in that window
would return a zero timestamp.

## Persistence

**Contract** — five commands, all arriving as messages from the local client and all
no-operations without a simulator:

- **change level** — hands the message to the simulator, which reads the destination and
  performs the transition; answers whether it will happen. Without a simulator, answers yes
  and does nothing, because a standalone level has nowhere to go and must not block.
- **save** — the simulator writes the whole world into the message.
- **load** — reads a save name from the message and asks the simulator to load it; without a
  simulator, falls back to the base session's load.
- **reload** — deliberately empty in single player.
- **switch distance** — sets the radius at which alife objects are promoted from offline to
  online. A single real read from the message.

## Restrictions

**Contract** — three commands that add, remove, or clear an entity's movement restrictors by
identifier, each carrying a restrictor kind (the volume is permitting or forbidding). All
are forwarded to the simulator and are no-operations without one, because a restrictor is a
property of the *server* record and must survive the entity going offline.

```text
FUNCTION add_restriction(message, entity_id)
  restrictor_id   = message.read_entity_id()
  restrictor_kind = message.read_enum()
  alife.add_restriction(entity_id, restrictor_id, restrictor_kind)
```

## `teleport_object`

**Contract** — moves an entity to a new place given as the full triple the alife simulation
needs: graph vertex, level vertex, and exact position. All three, because a position alone
does not tell the coarse simulation where the entity is and a graph vertex alone is not
precise enough to stand on.

## `sls_default`

**Contract** — asks the simulator to re-evaluate which objects should be online. Called when
the session wants the online set brought up to date immediately rather than at the
simulator's own cadence.

**Notes** — this one does *not* guard on the simulator existing, while its companion
predicate reports that a custom default exists only when there is one. The guard lives in the
caller; a rebuild should put it here instead.

## `level_name`

**Contract** — the level to load: the simulator's answer, which is where the player's saved
position says they are; without a simulator, the name parsed out of the session options.

## `on_death`

**Contract** — runs the base session's death handling, then tells the simulator. Order
matters: the base clears the entity's live state, and the simulator's handler may promote or
relocate the corpse and its inventory based on that cleared state.

## `restart_simulator`

**Contract** — replaces the running simulation with one loaded from a named save, without
leaving the session. This is how loading a save from inside a running game works.

```text
FUNCTION restart_simulator(save_name)
  options = alife.original_option_string      # read it back BEFORE destroying the simulator
  destroy alife
  server.clear_identifier_allocator()         # entity identifiers must restart from a
                                              # clean state or the save's ids collide

  persistent_game.params.game_or_spawn = save_name
  persistent_game.params.new_or_load   = "load"

  persistent_game.load_begin()
  alife = new AlifeSimulator(server, options)  # constructing it performs the load
  persistent_game.show_load_title("synchronising")
  device.precache(60 frames, blocking)
  persistent_game.load_end()
```

**Invariants** — the option string must be taken from the old simulator before it is
destroyed, and the identifier allocator must be cleared before the new one is built. Reverse
either and the new world either loses its options or is built on top of the old world's
identifier watermark, which produces entities whose identifiers do not match the ones in the
save.

The pre-cache of a fixed number of frames is a deliberate stall: it forces the renderer to
build every pipeline state and upload every texture the new world needs *before* the loading
screen comes down, so the first second of play does not hitch. Sixty frames is an empirical
figure, not a derived one.
