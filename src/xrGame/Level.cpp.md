# src/xrGame/Level.cpp

> The game layer's per-frame heartbeat: it owns the loaded level's managers, drains the network event queue, runs correction prediction, and drives every subsystem once per frame in a fixed order.

**Needs** — [`Level.h`](Level.h.md) · [`Level_Bullet_Manager.h`](Level_Bullet_Manager.h.md) · [`game_cl_base.h`](game_cl_base.h.md) · [`NET_Queue.h`](NET_Queue.h.md) · [`HUDManager.h`](HUDManager.h.md) · [`player_hud.h`](player_hud.h.md) · [`map_manager.h`](map_manager.h.md) · [`GametaskManager.h`](GametaskManager.h.md) · [`level_sounds.h`](level_sounds.h.md) · [`space_restriction_manager.h`](space_restriction_manager.h.md) · [`client_spawn_manager.h`](client_spawn_manager.h.md) · [`autosave_manager.h`](autosave_manager.h.md) · [`seniority_hierarchy_holder.h`](seniority_hierarchy_holder.h.md) · [`ai_space.h`](ai_space.h.md) · [`Actor.h`](Actor.h.md) · [`ParticlesObject.h`](ParticlesObject.h.md) · [`xrEngine/IGame_Level.h`](../xrEngine/IGame_Level.h.md) · [`xrPhysics/IPHWorld.h`](../xrPhysics/IPHWorld.h.md) · [`xrPhysics/PHCommander.h`](../xrPhysics/PHCommander.h.md) · [Seam: Script virtual machine](../../SYSTEM-REQUIREMENTS.md#seam-script-virtual-machine) · [Seam: Networking transport](../../SYSTEM-REQUIREMENTS.md#seam-networking-transport) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics) · [Seam: Compression](../../SYSTEM-REQUIREMENTS.md#seam-compression)
**Used by** — reached through its declarations in [`Level.h`](Level.h.md); callers name that, not this file.
**Tier floor** — T1: a fixed-rate frame loop with a hard tail, driving physics, script and network in one thread

## Purpose

The engine below provides a level as geometry, collision and a device. This file is the
*game's* level: the object that knows a match is in progress, owns every per-level manager,
and decides what happens in a frame and in what order.

It is split across several files (network start-up, save and load, demo recording, input);
this one carries the frame loop, the event drain, the correction prediction, the game clock
and the construction order. Read it as the answer to two questions: **what exists while a
level is loaded**, and **what happens each frame, in what order**.

The single most load-bearing thing on the page is the frame order. Nothing about it is
arbitrary — every adjacent pair has a reason, and several of the reasons are one-frame
latencies that are visible in gameplay.

## State

```text
RECORD Level
  # world
  map_data                : LevelMapSyncData    # name, version, checksum of the loaded level
  game                    : GameState           # the rules of the match, single or multiplayer
  server                  : optional<Server>    # present when this process is authoritative
  objects                 : object registry     # inherited from the engine's level
  bullet_manager          : BulletManager
  ph_commander            : PhysicsCommander    # deferred physics-step work, engine-owned
  ph_commander_scripts    : PhysicsCommander    # the same, script-owned; kept separate so a
                                                # script reload cannot drop engine work
  # per-level managers, absent on a dedicated server
  map_manager             : MapManager
  game_task_manager       : GameTaskManager
  level_sound_manager     : LevelSoundManager
  space_restriction_manager : SpaceRestrictionManager
  client_spawn_manager    : ClientSpawnManager
  autosave_manager        : AutosaveManager
  seniority_hierarchy_holder : SeniorityHierarchyHolder   # present even on a dedicated server
  # queues
  game_events             : time-ordered event queue
  game_spawn_queue        : queue<server record>
  # correction prediction
  objects_for_prediction  : list<GameObject>
  actors_for_prediction   : list<GameObject>
  need_prediction         : bool
  in_prediction           : bool
  prediction_steps        : int
  delta_update            : int (ms)    # smoothed interval between network updates
  last_net_update_time    : int (ms)
  # misc
  static_particles        : list<ParticlesObject>
  static_sounds           : list<sound>
  sound_registry          : map<text, sound>    # prefetched sounds, by normalized name
  current_control_entity  : optional<GameObject>
  feel_deny               : GlobalFeelTouch
```

**Invariants**
- Exactly one level object exists while a level is loaded, reachable globally. Most of the
  game layer reaches it that way rather than being handed it.
- A dedicated server constructs *no* presentation-side manager: no map, no tasks, no sounds,
  no restrictions, no spawn manager, no autosave. Every use of those is guarded.
- Server and client are not exclusive. Single player is a server *and* a client in one
  process. Demo playback inverts both: it reports client and not server even when a server
  object exists.
- Correction prediction may not nest: the in-prediction flag is what the rest of the game
  reads to know it must not spawn, destroy or send.

## Construction

**Contract** — builds every per-level manager and attaches the level's five named engine
event handlers. Order within construction is mostly free; what is not free is the
dedicated-server split and the fact that the physics commanders and the bullet manager exist
on every configuration, because both are simulation rather than presentation.

**Notes** — the first-person presentation is constructed here and loads its default
configuration immediately, before any entity exists. That is why a weapon can be activated
on the very first frame after a load.

The smoothed network-update interval is seeded with the fixed physics timestep expressed in
milliseconds, so interpolation is sane before a single update has arrived.

## Destruction

**Contract** — teardown in a deliberate order, and the order *is* the contract. The
first-person presentation and the head-up display go first, before anything they might
reference; the physics world next, so no object outlives the bodies it owns; then particles,
then sounds, then the managers, then the script process, then the game state, the queues and
the bullet manager, and last the artificial-intelligence space.

**Invariants** — the script process must be removed before the script-owned physics
commander is released, or deferred script work would run against a dead interpreter.

**Notes** — the shared default trade parameters are cleared here, explicitly because no
better place was found: they must be rebuilt per saved game, and the level's destruction is
the only point that reliably happens. A rebuild should own them per session instead.

The tutorial sequencer's stored input receiver is cleared if it points at this level — the
tutorial outlives the level and would otherwise deliver input into freed memory.

## `OnFrame`

**Contract** — the game layer's frame. It runs after the engine has advanced the device and
before the render. The order below is the load-bearing content of this file.

```text
FUNCTION on_frame()
  update the global touch-denial sense volume
  set "objects as crows" rendering off in single player, on otherwise

  # 1. commit last frame's bullets BEFORE anything reads the world
  bullet_manager.commit_events()

  # 2. network: if the connection is gone, tear down and leave
  IF disconnected
    IF this is a multiplayer client THEN clear all objects
    defer the kernel disconnect; RETURN
  ELSE
    client_receive()          # drains the transport into the event queue

  # 3. drain the event queue up to (server time - latency)
  process_game_events()

  # 4. correction prediction, if an update arrived that needs it
  IF need_prediction THEN make_net_correction_prediction()

  # 5. presentation-side managers (never on a dedicated server)
  map_manager.update()                  # on a worker thread if configured
  IF single player AND precache is finished
    game_task_manager.update_tasks()

  # 6. the engine's own frame: every registered object's update
  base.on_frame()

  # 7. hand the clock to the weather system
  environment.set_game_time(day_time_seconds, time_factor)

  # 8. the level's script process
  script_process.update()

  # 9. deferred physics work, engine's then scripts'
  ph_commander.update()
  ph_commander_scripts.update()

  # 10. bullets again: build this frame's render set, AFTER objects moved
  bullet_manager.commit_render_set()

  # 11. ambient sound, then one step of script garbage collection
  level_sound_manager.update()          # on a worker thread if configured
  script_gc()                           # likewise
```

**Invariants** — the bullet manager is touched **twice**, and the two halves must straddle
the object update. Committing events first applies last frame's hits before anybody reads
their own health; building the render set last captures where the tracers actually ended up
after everything moved. Merging them into one call changes what the player sees.

Task updates are gated on the precache being finished, because a task update can show a
message and messages must not appear over the loading screen.

**Notes** — four pieces of work can be pushed onto a parallel queue instead of run inline,
each behind its own configuration flag: the map manager, the ambient sound manager, script
garbage collection, and (disabled) the task manager. They are the four that touch no shared
mutable state. That is the extent of this frame's parallelism — everything else is serial by
construction, and that is the engine's defining performance characteristic.

## `script_gc`

**Contract** — one incremental step of the script interpreter's garbage collector per frame,
by one of two strategies: a time-budgeted collection where the interpreter supports it, or a
fixed step count. The time-budgeted strategy **demotes itself permanently** the first time
the interpreter reports it unsupported, so the probe costs one call rather than one per
frame.

**Notes** — running collection incrementally every frame rather than letting it run to
completion is the decision: a full collection is a visible hitch, and the script heap is
large enough that one would happen regularly.

## `ProcessGameEvents`

**Contract** — drains the ordered event queue of everything whose time has come, where "come"
means **server time minus a fixed latency allowance**. That deliberate lag is what lets
events from a server arrive out of order and still be applied in order.

```text
FUNCTION process_game_events()
  deadline = server_time - latency_allowance
  WHILE queue has an event at or before deadline
    take (id, destination, type, payload)
    SWITCH id
      spawn         -> create the object described by the payload
      event         -> deliver to one object (see below)
      move_players  -> teleport each listed actor, then acknowledge
      statistics    -> feed the weapon-usage statistics (multiplayer only)
      file_transfer -> feed the transfer session, if one is still open
      game_message  -> hand to the game rules
  IF authoritative AND multiplayer
    let the statistics subsystem ask its clients for a check
```

**Invariants** — the teleport batch must be acknowledged, because the server withholds
further movement until it is.

## `cl_Process_Event`

**Contract** — delivers one event to one object by identity. An unknown destination or a
destination that is not a game object is dropped with a diagnostic, never a failure — events
routinely arrive for objects that have already gone away.

The destroy-and-reject event is the one that is not a plain delivery:

```text
FUNCTION cl_process_event(destination, type, payload)
  target = objects.find(destination)
  IF target IS none OR NOT a game object
    log; RETURN

  IF type IS NOT destroy_reject
    IF type IS destroy THEN tell the game rules first
    target.on_event(payload, type)
    RETURN

  # destroy_reject: one message meaning two things — the holder gives up
  # ownership, and the held object is destroyed. They must happen in that
  # order or the holder would release an already-dead object.
  peek the held object's identity out of the payload without consuming it
  target.on_event(payload, ownership_reject)
  IF the held object still exists
    tell the game rules, then deliver destroy to it
```

**Notes** — the payload is peeked and rewound rather than consumed, because the reject
handler needs to read the same identity from the start. That is a framing decision worth
keeping: one message carries one identity read by two handlers.

## `make_NetCorrectionPrediction`

**Contract** — reconciles a received network update with local physics. A received update
describes the world as it was some steps ago; this re-runs those steps locally so the object
ends up where it should be *now*, and optionally a further run of steps into the future so
interpolation has a target. Runs the physics world in a frozen mode throughout, so the
prediction does not emit collisions, sounds or events into the game.

```text
FUNCTION make_net_correction_prediction()
  need_prediction = false; in_prediction = true
  saved_step_count = physics.steps_taken
  physics.steps_taken = physics.steps_taken - prediction_steps
  clamp prediction_steps to 10 when the diagnostic flag is on
  physics.freeze()

  FOR EACH object IN objects_for_prediction
    object.before_prediction()          # install the received state

  FOR EACH step IN 1 .. prediction_steps
    physics.step()
    FOR EACH actor IN actors_for_prediction NOT yet activated
      actor.before_prediction()         # an actor joins mid-run, at its own step

  FOR EACH object IN objects_for_prediction
    object.in_prediction()              # this is "now"

  IF interpolation is enabled
    FOR EACH step IN 1 .. interpolation_steps
      physics.step()                    # run on into the future
    FOR EACH object IN objects_for_prediction
      object.after_prediction()         # this is the interpolation target

  physics.unfreeze()
  physics.steps_taken = saved_step_count
  prediction_steps = 0; in_prediction = false
  clear both prediction lists
```

**Invariants** — the physics step counter is rewound before the run and restored after, so
that anything keyed to "how many steps has the world taken" is not disturbed by a
prediction. Freezing is what keeps the run side-effect-free.

**Notes** — the actors are stepped into the run *lazily*, each one activating on the step
where its own update was received, rather than all at the start. That is why they are a
separate list from the general objects.

The ten-step clamp applies only when the diagnostic flag is on, so the release build will
happily run an unbounded number of prediction steps after a long stall. The million-step
guard in the step setter is an assertion, not a clamp.

## `UpdateDeltaUpd` / `ReculcInterpolationSteps` / `GetInterpolationSteps` / `InterpolationDisabled` / `SetNumCrSteps`

**Contract** — the interpolation budget. The interval between network updates is smoothed
and converted into a number of physics steps to interpolate over. Three configurations:

- A configured interpolation *time* above zero: steps are that time divided by the physics
  timestep, and the measured interval is ignored.
- Exactly zero: derive the step count from the measured interval, clamped to between 3 and
  60 steps.
- Below zero: interpolation is off entirely and the future-prediction half is skipped.

**Invariants** — the smoothing is asymmetric on purpose: an interval *shorter* than the
current estimate is blended in at one part in eleven, while a longer one replaces the
estimate outright. Latency is allowed to rise instantly and must fall slowly, because an
optimistic estimate produces visible rubber-banding and a pessimistic one only produces lag.

**Notes** — the clamp floor of 3 steps and ceiling of 60 are not derived from anything
stated. The ceiling is one second at the fixed timestep; the floor is not explained.

## `AddObject_To_Objects4CrPr` / `AddActor_To_Actors4CrPr` / `RemoveObject_From_4CrPr`

**Contract** — registration for the next prediction run. Both adders are idempotent by
linear search. Removal takes an object out of both lists, and is called whenever an object
gains a parent — a carried object must not be predicted, because prediction assumes a free
body and would corrupt one that is attached.

## `OnRender`

**Contract** — the game layer's draw, bracketed by the renderer's world-render hooks. Order:
the engine's scene, then the game rules' own overlay, then bullet tracers, then the world
ends, then the head-up display. Tracers are drawn inside the world bracket and the interface
outside it, which is what makes tracers fog and shadow like geometry while the interface does
not.

**Notes** — the head-up display is suppressed while the external screenshot tool is active,
so a captured image has no interface in it.

Everything past that point is diagnostics: navigation-graph overlays, per-object debug draw,
vision cones, visibility rays, skeletons within twenty metres, network statistics. None of it
exists in a shipped build, but the *set* of things the engine's authors found worth drawing
is a fair list of what a rebuild will also want to see.

## `OnEvent`

**Contract** — the five named engine events this level handles. Only two do anything: a
spawn request by name, which is the console's entity-spawn command, and a demo playback
request, which attaches a camera effector reading the named file. The music and environment
events are registered and empty; a rebuild should not attach them.

**Notes** — the spawn event parses a name out of a text parameter with a scanner that stops
at whitespace, so an entity section containing a space cannot be spawned from the console.

## The game clock

**Contract** — the level exposes the match's clock in several shapes, all derived from the
game rules' own time, which is the authority:

- **game time** — absolute milliseconds since the campaign's epoch, 64-bit. This is what
  saves and the alife simulation use.
- **start game time** — the epoch.
- **environment game time** — a *separate* clock for the weather system, which can run at a
  different rate from the game's own. That separation is what lets a script accelerate the
  sky without accelerating the simulation.
- **day time** — the hour, 0 through 23.
- **day time in milliseconds / in seconds** — the time within the current day, obtained by
  taking the whole clock modulo one day.
- **calendar breakdown** — year, month, day, hour, minute, second, millisecond.

Each clock has a settable rate multiplier, and the environment's can be set with an explicit
"from this game time onward" so a rate change does not retroactively move the sky.

**Invariants** — the environment readers must tolerate the game rules not existing yet and
answer zero, because the weather system is asked for the time during level loading.

**Notes** — the day-length constant is composed as twenty-four hours of sixty minutes of
sixty seconds of a thousand milliseconds, which fixes an in-game day to exactly one real day
of game-clock time. The *rate* is what makes a day pass in minutes.

## `IsServer` / `IsClient`

**Contract** — the authority test, and it is not a partition. A process is a server when it
owns a server object and is not replaying a demo. It is a client when it is replaying a
demo, or when it owns no server object. A single-player process is therefore **both**, and
demo playback is a client that also has a server.

**Invariants** — the demo-playback special case in both is what makes a recorded session
replay as if received from the network.

## `PrefetchSound`

**Contract** — loads a sound ahead of time and keeps it in a registry keyed by *normalized*
name: lowercased, extension stripped. Already-registered names are not reloaded. The whole
registry is dropped at level teardown.

**Invariants** — normalization is what makes the cache work at all, since the same sound is
referenced from configuration with varying case and with or without an extension.

## `ClearAllObjects` / `remove_objects`

**Contract** — drops every object in the level. Called when a multiplayer client loses its
connection, where the alternative is a frozen world full of stale entities.

## `OnAlifeSimulatorLoaded` / `OnAlifeSimulatorUnLoaded`

**Contract** — both reset the map-marker storage and the task storage, and they do the same
thing. Those two subsystems index by alife identity, so their contents are meaningless
across a change of simulation in either direction.

## `spawn_item`

**Contract** — creates an entity from a configuration section at a position and navigation
vertex, optionally parented, and either sends it into the world immediately or returns the
server record for the caller to finish. This is *the* programmatic spawn path — scripts,
starting equipment and scripted rewards all reach the world through it.

## `GetLevelInfo` / `name` / `version` / `IsChecksumsEqual`

**Contract** — the loaded level's identity: its name, the authored version string, and a
checksum comparison used to refuse a client whose copy of the level differs from the
server's. The checksum is over the level's own data, which is why a modified level cannot
join an unmodified server.

## `PhisStepsCallback`

**Contract** — the per-physics-step hook, and it is empty. It was a per-actor position
history for multiplayer lag compensation, removed as too slow. Its continued existence as a
registered no-op is a cost with no benefit; the *idea* — that lag compensation needs a
position history sampled at physics rate, not at frame rate — is worth keeping.

## `MakeReconnect`

**Contract** — defers a disconnect followed by a start with the same server and client
option strings, so a connection can be rebuilt without the caller knowing how it was made.
Guarded against queueing twice.

**Notes** — the option strings are duplicated onto the heap and handed to the deferred event
as integers. Nothing frees them; a rebuild passing owned strings through an event queue must
decide who owns them.

## `get_RPID`

**Contract** — returns "no such respawn point", always. The body that looked one up by name
is commented out. Nothing depends on the answer.

## `CZoneList`

**Contract** — a touch-sensing filter that admits an object only when its configuration
section is in an explicit set *and*, if it is an anomaly, the anomaly is currently enabled.
It is how the first-person display knows which anomalies are near enough to register on a
detector: the detector declares the kinds it can see, and the sense volume does the rest.

**Notes** — it lives in this file rather than with the detectors because the level owns the
single instance. That is arbitrary and a rebuild should move it.

## `DumpStatistics`

**Contract** — the performance overlay: client send and receive timings, compression, the
bullet-manager commit, then the AI budget broken into thinking, range queries, pathfinding,
node queries, vision (split into portal traversal and ray casts) and script collection, plus
the script heap size. The breakdown names exactly the costs the authors expected to dominate,
and a rebuild will want the same eight counters.
