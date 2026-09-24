# src/xrGame/alife_simulator.cpp

> The alife simulation's top object: constructing it starts a game, destroying it ends one.

**Needs** — [`alife_simulator.h`](alife_simulator.h.md) · [`alife_update_manager.h`](alife_update_manager.h.md) · [`alife_interaction_manager.h`](alife_interaction_manager.h.md) · [`alife_simulator_base.h`](alife_simulator_base.h.md) · [`alife_object_registry.h`](alife_object_registry.h.md) · [`ai_space.h`](ai_space.h.md) · [`object_factory.h`](../xrServerEntities/object_factory.h.md) · [`xrEngine/IGame_Persistent.h`](../xrEngine/IGame_Persistent.h.md) · [Seam: Script virtual machine](../../SYSTEM-REQUIREMENTS.md#seam-script-virtual-machine)
**Used by** — reached through its declarations in [`alife_simulator.h`](alife_simulator.h.md); callers name that, not this file.
**Tier floor** — T2: startup sequencing, a small cache of open files

## Purpose

`CALifeSimulator` is the concrete assembly of everything the alife layer is: the
registries, the graph, the time and switch managers, the update loop and the offline
interaction rules, all inherited into one object. This file holds almost none of that
behaviour. What it holds is the **startup order** — the sequence of things that must
happen, in this order, for a game to exist — and a small cache of configuration files.

There is exactly one alife simulator at a time, and its lifetime *is* the session's: it is
constructed when a new or saved game starts and destroyed when the player returns to the
menu. A rebuild should keep that identity, because a great deal of code tests "is there an
alife simulation" to distinguish single player from multiplayer.

## State

```text
RECORD ALifeSimulator                # plus everything it inherits
  config_cache : list<(name, open reader)>   # most-recently-used first, never evicted
```

The cache holds *open handles* to configuration files, not parsed content, and it is
ordered by recency. Nothing ever removes an entry: the list grows to the number of
distinct configuration files the session asked for and is closed wholesale at
destruction. That is the honest description — it is a lookup table with recency ordering,
not a bounded cache, despite being named one.

## Construction — the startup order

**Contract** — builds the whole alife layer and loads a game. Fails hard on invalid
session parameters or a missing script entry point. Blocking, and by far the longest
operation in a session's life. Takes the session's command line by reference and
**rewrites it** before returning.

```text
FUNCTION construct(server, command_line)
  # 1. Bases first: the registries, graph, time, switch and update machinery.
  construct base layers with (server, configuration section "alife")

  # 2. The script world is torn down and rebuilt, unless the session asked otherwise.
  IF command line does not request keeping the script engine
    restart the script virtual machine

  # 3. Publish ourselves before anything can look for us.
  ai_space.alife = this

  # 4. Interpret the session parameters.
  setup_command_line(command_line)
  params = the persistent game parameters
  REQUIRE params.spawn_or_save is non-empty
      AND params.alife_mode == "alife"
      AND params.game_type  == "single"
    ELSE FAIL WITH invalid server options

  # 5. Rewrite the command line into the canonical "<spawn_or_save>/<type>/<mode>" form.
  command_line = join(params.spawn_or_save, params.game_type, params.alife_mode)

  # 6. Hand control to the script layer's start-game entry point, telling it whether
  #    this is a new game.
  callback_name = configuration("alife", "start_game_callback")
  REQUIRE that script function exists   ELSE FAIL WITH missing start game callback
  call it with (is_new_game)

  # 7. Load the world: either a level's spawn file or a saved game.
  load(params.spawn_or_save,
       is_save  = params.new_or_load == "load" ? false : true,
       is_new   = params.new_or_load == "new")
```

**Invariants** — the order of steps 2, 3 and 6 is the load-bearing part.

- The script engine is restarted **before** the simulator publishes itself, because the
  restart discards every script-held reference and a stale reference to a previous
  session's simulator is exactly the failure this prevents. The escape hatch — keep the
  engine — exists for development, where reloading a game without losing script debugging
  state is worth the risk.
- The simulator is published **before** the start-game callback runs, because that
  callback is script code that immediately reaches for the simulation.
- The start-game callback runs **before** anything is loaded, so the script layer can
  install its own state and its own callbacks before the first entity exists.
- The session is refused rather than adapted if it is not a single-player alife session:
  the multiplayer server builds a different simulator entirely.

**Notes** — two comparisons in the original are written against the wrong sentinel, so
`is_new_game` and the `is_save` argument are constant in practice. Neither has an
observable effect in the shipped flow, because the third argument (`is_new`) carries the
same distinction correctly and is the one the loader branches on. A rebuild should pass
one clear flag rather than three overlapping ones.

The command-line rewrite is a side effect on the caller's string. It exists so that the
console's saved "restart the last session" command reproduces the session. A rebuild
should return the canonical form rather than mutate an argument.

## `destroy`

**Contract** — tears the simulation down and unpublishes it. Must be called before
destruction; the destructor asserts that it was.

```text
FUNCTION destroy()
  update_manager.destroy()        # stop the loop, release the world
  REQUIRE ai_space.alife is still this
  ai_space.alife = none
```

**Invariants** — the two-step *destroy then release* is not optional. Tearing down the
world runs entity unregister hooks, script callbacks and registry sweeps, all of which
reach the simulator through the published global; unpublishing first would strand them.
This is the same two-phase shape the object registry's destruction uses, for the same
reason, and it is the rule conformance §6 states as "a destroyed entity is unreferenced by
the scheduler, the render graph and the physics world before its memory is released".

The destructor itself only closes the cached configuration files, and asserts that
`destroy` already ran.

## `get_config`

**Contract** — yields an open reader over a configuration file under the game's
configuration root, by relative name. Returns nothing if the file does not exist. Never
closes what it opens; every handle is closed when the simulator is destroyed. Not
thread-safe. Callers must not close the returned handle.

```text
FUNCTION get_config(name) -> optional<Reader>
  IF name is in the cache
    move its entry to the front       # recency ordering
    RETURN its reader
  path = resolve "$game_config$" + name
  IF path does not exist -> RETURN none
  reader = open(path)
  insert (name, reader) at the front
  RETURN reader
```

**Notes** — the lifetime rule is the reason this exists at all: scripts and entity code
ask for the same handful of configuration files repeatedly over a session, each time
wanting a reader they will not own. Tying every handle's lifetime to the session removes
the ownership question entirely. The cost is that the handles stay open for the session;
the count is small and bounded by the authored data.

The recency reordering does nothing useful — with no eviction, order affects only how
quickly a linear search finds an entry, and the list is short. A rebuild should use a map
keyed by name and drop the ordering.

## `setup_simulator`

**Contract** — stamps an alife server object with a back-reference to this simulator.
Called for every object as it is created, so that an entity can reach the simulation
without going through the global.

## `object_exists_in_alife_registry`

**Contract** — a free query: does an entity with this identifier exist in the alife object
registry? False when there is no simulation at all, which is the multiplayer answer.

**Notes** — it exists as a free function rather than a method so that code which must
compile without knowing about the simulator (the entity-definition module shared with the
tools) can still ask. A rebuild without the shared-module constraint folds it into the
registry.

## `reload`

**Contract** — re-reads the simulation's tuning from a configuration section. Delegates
to the update manager. Exists for development: changing alife parameters without
restarting the session.
