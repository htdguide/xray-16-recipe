# src/xrGame/alife_update_manager.cpp

> The alife simulation's heartbeat: the one place per frame that decides which offline entities come online, advances the offline world under a time budget, and performs the two world-scale operations — starting a new game and changing level — that rewrite everything else.

**Needs** — [`alife_update_manager.h`](alife_update_manager.h.md) · [`alife_simulator_header.h`](alife_simulator_header.h.md) · [`alife_time_manager.h`](alife_time_manager.h.md) · [`alife_graph_registry.h`](alife_graph_registry.h.md) · [`alife_schedule_registry.h`](alife_schedule_registry.h.md) · [`alife_spawn_registry.h`](alife_spawn_registry.h.md) · [`alife_object_registry.h`](alife_object_registry.h.md) · [`alife_switch_manager.h`](alife_switch_manager.h.md) · [`alife_storage_manager.h`](alife_storage_manager.h.md) · [`ef_storage.h`](ef_storage.h.md) · [`xrServer.h`](xrServer.h.md) · [`Level.h`](Level.h.md) · [`mt_config.h`](mt_config.h.md) · [`restriction_space.h`](../xrServerEntities/restriction_space.h.md) · [`xrAICore/Navigation/graph_engine.h`](../xrAICore/Navigation/graph_engine.h.md) · [`xrEngine/IGame_Persistent.h`](../xrEngine/IGame_Persistent.h.md) · [`xrEngine/profiler.h`](../xrEngine/profiler.h.md) · [`xrNetServer/NET_Messages.h`](../xrNetServer/NET_Messages.h.md) · [Seam: Script virtual machine](../../SYSTEM-REQUIREMENTS.md#seam-script-virtual-machine)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: registry orchestration and one byte-exact save round trip, which is delegated; the only hard constraint is that the whole thing fits in a declared microsecond budget

## Purpose

The alife simulation is a pile of registries — objects, graph positions, schedules,
spawns, time, surges, storage — and this file is the only thing that *drives* them. It is
the top of the inheritance pile that makes up the simulator, and it contributes exactly
three things:

1. a **per-frame tick**, registered with the engine's scheduler like any other scheduled
   object, that runs the two halves of the simulation (see `update`);
2. the **world-scale lifecycle operations** — create a new game, load a save, change
   level — each of which unloads and rebuilds the entire object registry;
3. the **script-facing verbs** that mutate an entity's alife state from outside: teleport
   it, forbid it from switching online, attach or drop a restrictor.

Group (3) is a grab bag with no algorithm in it, and it lives here for one reason: these
are the operations that must be performed on the *server object* rather than on a live
client object, whether or not the entity is currently online. Nothing else in the game has
a handle that reaches an offline entity.

## State

```text
RECORD UpdateManagerTuning            # all four read from one configuration section
  max_process_time        : int       # microseconds the offline simulation may consume per tick
  update_monster_factor   : real      # fraction of that budget reserved for offline creature movement
  objects_per_update      : int       # how many scheduled records the offline pass advances per tick
  schedule_min            : int       # the scheduler bounds this object declares for itself
  schedule_max            : int

RECORD UpdateManagerState
  first_time              : bool      # the very first tick never runs off the main thread
  changing_level          : bool      # latched for the duration of a level change; re-entry is refused
```

Invariants:

- `max_process_time` and `objects_per_update` are **not** applied where they are read.
  They are pushed down into the graph registry and the schedule registry respectively, at
  construction and again on every reload, because those are the two components that
  actually spend the budget. A rebuild that stores them here and lets the consumers read
  them back has the same behaviour; the hazard is forgetting the re-push on reload.
- the level change is **not re-entrant**. A second request while one is in flight is
  dropped silently, because the first has already begun rewriting the actor's position.

## `update`

**Contract** — one tick of the whole offline world. Two halves, in a fixed order, and the
order is load-bearing: *switch first, advance second.*

```text
FUNCTION update()
  update_switch()          # who crosses the online/offline boundary this tick
  update_scheduled(false)  # advance the offline records that are due
```

**Invariants** — the evaluation storage the switch decisions read must be told it is
serving the alife simulation and not a live creature's brain before either half runs;
`update_switch` does that, and `update_scheduled` is told not to repeat it.

**Notes** — the switch pass walks *only the entities registered on the game-graph vertices
near the loaded level*, not the whole world, and it stamps each one with the current cycle
count so that an entity reachable from two vertices is considered once. During the
precache spin (the frame budget the level spends warming its working set before the player
gets control) the pass is told to run to completion rather than under its usual budget,
because everything the level will show must already be online when the spin starts.

## `shedule_Update`

**Contract** — the entry point the engine's scheduler calls. Declares itself always needed
and reports a fixed importance of one half, which places it permanently in the middle of
the scheduler's priority ordering rather than letting it compete for attention.

```text
FUNCTION shedule_Update(elapsed)
  advance the base scheduled bookkeeping
  IF the simulator is not yet initialized THEN RETURN

  IF this is not the first tick AND the build is configured to run alife off-thread THEN
    enqueue update() into the frame's parallel work list
    RETURN                                  # it runs beside the render phase, not before it
  mark first tick done
  update()
```

**Notes** — the first tick is forced onto the main thread. The reason is ordering, not
safety: on the very first tick the simulator is still finishing its own bring-up, and the
level's load sequence is entitled to observe a fully-switched world before it hands
control to the player. Every subsequent tick may run beside the render phase because the
engine is not pipelined — the simulation has already finished for the frame and the only
thing the alife pass can race with is drawing, which never reads a server object.

The destructor must remove the enqueued closure from the frame's parallel list as well as
unregistering from the scheduler. Two registrations, two removals: a rebuild that forgets
the second one runs a tick against a freed simulator.

## `new_game`

**Contract** — build a world from the shipped spawn file rather than from a save. Does not
return a result; a failure here is fatal.

```text
FUNCTION new_game(save_name)
  show the "creating new game" title on the loading screen
  unload()                                # discard any previous world completely
  reload(configuration section)           # re-read tuning, re-push the budgets
  spawns().load(save_name)                # the level set's authored spawn records
  graph().on_load()
  server identifier generator reset to zero
  time_manager().init(section)            # the in-world clock starts at its authored value

  REQUIRE object registration is currently permitted
  forbid object registration
  spawn_new_objects()                     # instantiate every authored spawn record
  permit object registration
  FOR EACH object IN objects()
    object.on_register()
```

**Invariants** — registration is *deliberately suppressed* while the spawn records are
instantiated, and every object's registration hook is run afterwards in one sweep. The
reason is that a spawn record may reference another entity by identifier — a trader's
inventory, a creature's smart terrain, a squad's members — and a hook that runs while the
world is half-built sees dangling references. Two phases: create everything, then let
everything look around.

## `load`

**Contract** — bring a world into existence, from a named save if one exists and from the
spawn file otherwise. The single entry point for both cases, which is why the "no save
found" path is not an error unless the caller said it must be one.

```text
FUNCTION load(game_name, no_assert, new_only)
  show the "loading alife simulator" title
  remember game_name as the last saved game        # the quick-load target
  IF new_only OR the storage manager cannot load game_name THEN
    REQUIRE new_only OR (no_assert AND game_name is non-empty)
    new_game(game_name)
  IF a level is already up THEN tell it the simulator is loaded
  show the "connecting" title with the level's name
```

**Notes** — the closing title change is not cosmetic bookkeeping: single player is a
client connecting to a server in the same process, and this is the point at which the
alife side is ready and the client side has not yet joined. The loading screen tells the
truth about the architecture.

## `load_game`

**Contract** — validate that a named save exists and rewrite the server's command line to
name it. Returns whether the save was found; refuses to assert only when told to.

```text
FUNCTION load_game(game_name, no_assert) -> bool
  IF no file exists for game_name under either the current or the legacy save extension THEN
    REQUIRE no_assert
    RETURN false
  splice game_name into the server command line, before its first '/'
  RETURN true
```

**Notes** — two save extensions are accepted because two generations of the game shipped
with different ones, and a player's existing saves must keep loading. The command-line
rewrite is how a save name reaches the server: the session is *started* by a server
command line of the form `<save or level>/<options>`, and loading a save means replacing
the part before the slash. A rebuild with a real session-parameters record should carry
one; the shape to preserve is that the save name and the session options travel together.

## `change_level`

**Contract** — move the actor to another level. Returns whether the change was accepted;
refuses if one is already in progress. Blocks for the duration of an autosave.

This is the most delicate function in the alife layer, and the delicacy is entirely about
*which position gets written to the save*.

```text
FUNCTION change_level(packet) -> bool
  IF already changing level THEN RETURN false
  call the script hook "_G.CALifeUpdateManager__on_before_change_level" with the packet

  Level().ClientSend()            # pull the live client's state up into the server objects
  mark changing level

  remember the actor's current graph vertex, level vertex, position, angles and torso
  IF the actor is inside a holder (a vehicle) THEN remember the holder's four as well

  read the destination graph vertex, level vertex, position and angles out of the packet
  and write them directly onto the actor's server object

  Level().ClientSave()            # the client serializes itself against the NEW position
  derive the actor's torso rotation from the new angles (roll forced to zero)
  IF the actor is inside a holder THEN move the holder to the actor's new position too

  save to "<user name> - autosave", and splice that name into the server command line
  restore the actor's remembered five values, and the holder's remembered four
  RETURN true
```

**Invariants** — after the call the in-memory world is **exactly as it was**; only the
save file on disk describes the new level. That is the whole trick. The engine then
restarts the session from that save, which is what actually performs the level change.
Saving a position the running world does not have is the reason the values are swapped in
and out around the save rather than simply assigned.

**Notes** — the comment in the source is worth preserving as a decision: the usual
"prepare every object for saving" path cannot be used here, because the order must be
*collect the client's updates → move the actor's server object → let the client serialize
itself → put the actor back*. A generic prepare-for-save runs the first and third steps
together and there is no seam between them to move the actor in.

The holder is moved with the actor because a player who changes level while driving must
arrive still driving, and the vehicle's position is authoritative for the pair.

## `jump_to_level`

**Contract** — a debug and script verb: choose a destination game-graph vertex on a named
level and request a level change to it. Silent no-op if the level has no vertices at all.

```text
FUNCTION jump_to_level(level_name) -> none
  search the game graph from the actor's vertex for any vertex on that level
  IF the search succeeded THEN
    destination = the vertex it selected
  ELSE
    # unreachable by travel: fall back to nearest by straight-line distance
    destination = the vertex on that level whose game point is closest to the actor's
    IF there is no such vertex THEN report and RETURN
  send a change-level message carrying the destination vertex, its level vertex and
  its level point, with a zero orientation
```

**Notes** — the fallback matters. The game graph is not fully connected in the shipped
data (some levels are reachable only through a scripted transition), so a pathfinding
search from the actor to an arbitrary level routinely fails. Refusing to jump in that case
would make the verb useless exactly where a developer needs it; picking the geometrically
nearest vertex is wrong as *travel* and right as *teleport*, which is what was asked for.

## `teleport_object`

**Contract** — move any entity, online or not, to a given game vertex, level vertex and
position. Logs and returns if no such entity exists.

```text
FUNCTION teleport_object(id, game_vertex, level_vertex, position)
  object = objects().object(id, tolerate_missing)
  IF none THEN report and RETURN
  IF the object is online THEN switch it offline first
  move it between game-graph vertices in the graph registry
  set its level vertex and position
  IF it is a creature THEN set its next graph vertex to its current one
```

**Invariants** — an online entity is demoted before it is moved. Moving a live client
object by rewriting its server record would leave the two sides disagreeing about where
the entity is, and the promotion path is the only thing that reconciles them.

The last step cancels any offline travel in progress: a creature that was walking from
vertex A to vertex B and is teleported to C must not continue arriving at B.

## `add_restriction`, `remove_restriction`, `remove_all_restrictions`

**Contract** — attach or detach a restrictor volume to a creature's *dynamic* restriction
set, as an in-restriction (it may only go here) or an out-restriction (it may not go
here). Every failure is a logged no-op, never an error: the caller is a script, and a
script naming a dead entity must not take the game down.

**Invariants** — four things are checked before anything is written, and each has its own
message because the script author needs to know which one failed: the entity exists, the
restrictor exists, the entity is a creature, and the restrictor is actually a space
restrictor. Adding the same restrictor twice is checked in development builds only and is
treated as a caller bug.

**Notes** — "dynamic" distinguishes these from the restrictions the entity's configuration
section gives it, which are fixed. The two sets are unioned by the pathfinder's cost
model. An entity's effective movement space is the intersection of its in-restrictors
minus the union of its out-restrictors — that rule lives with the restrictor system, not
here; this file only maintains the lists.

## `set_switch_online`, `set_switch_offline`, `set_interactive`

**Contract** — three one-line setters on a server object, reached by entity identifier.
The first two veto the corresponding transition for that entity — a scripted sequence uses
them to pin a character online while it is being talked to, or to keep a distant entity
from ever materializing. The third marks an entity as one the player may interact with.

**Notes** — these are vetoes, not commands. Clearing the online veto does not bring an
entity online; it only stops the switch pass from refusing to.

## `set_process_time`, `objects_per_update`, `reload`, `init_ef_storage`

**Contract** — the tuning path. `reload` re-reads the configuration section and pushes both
budgets down to their owners; `set_process_time` converts the configured microsecond
budget into the fraction the offline movement simulation may have; `objects_per_update`
forwards to the schedule registry; `init_ef_storage` puts the shared evaluation storage
into alife mode before any offline decision reads it.

**Notes** — the budget split subtracts the creature-movement fraction from the total in a
single expression whose units do not agree with themselves: the configured value is in
microseconds, the factor is a fraction, and the divisor converting to seconds is applied
to only one of the two terms. **This is not recoverable as an intention.** The resulting
number is what the shipped configuration was tuned against, so a rebuild that "fixes" the
arithmetic changes how much offline work happens per tick on the shipped data. Reproduce
the expression, not the apparent intent.
