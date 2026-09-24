# src/xrEngine/IGame_Level.cpp

> The level's lifecycle: load its geometry, collision and objects in the one order that works; tear it down; and distribute every emitted sound to the entities that can hear it.

**Needs** — [`IGame_Level.h`](IGame_Level.h.md) · [`IGame_Persistent.h`](IGame_Persistent.h.md) · [`xr_object_list.h`](xr_object_list.h.md) · [`CameraManager.h`](CameraManager.h.md) · [`CustomHUD.h`](CustomHUD.h.md) · [`Feel_Sound.h`](Feel_Sound.h.md) · [`Environment.h`](Environment.h.md) · [`Render.h`](Render.h.md) · [`xrCDB/xr_area.h`](../xrCDB/xr_area.h.md) · [`Common/LevelStructure.hpp`](../Common/LevelStructure.hpp.md) · [Seam: Static collision database](../../SYSTEM-REQUIREMENTS.md#seam-static-collision-database) · [Seam: Audio device](../../SYSTEM-REQUIREMENTS.md#seam-audio-device) · [Level data format](../../SYSTEM-REQUIREMENTS.md#5-data-and-persistence)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: it reads a chunked binary level file and hands the collision database raw triangle arrays.

## Purpose

A level is the unit of loading in this engine: one map's geometry, its collision database, its visibility topology, its spatial index, its objects and its sound scene, all created together and destroyed together. Exactly one level exists at a time, reachable from a single process-wide reference, and the fact that it is a *single* global is the design — everything from the AI to the renderer reaches the world through it.

This file owns the sequence. Its other job, which sits here because it needs both the spatial index and the sound scene, is turning an emitted sound into a perception event for every creature in range.

## State

```text
RECORD Level
  objects            : ObjectList        # every client object; see xr_object_list.cpp
  object_space       : CollisionArea     # static collision database + visibility topology
  cameras            : CameraManager
  sound_scene        : SoundScene
  config             : ConfigFile        # the level's own "level.ltx"
  hud                : HudInterface
  ready              : bool              # invariant: false until load completes, false again at stop
  current_entity     : optional<entity>  # who the player controls
  current_view_entity: optional<entity>  # whose eyes the camera uses; usually the same
  pending_sound_events : list<(listener, sound, power)>
  ambient_sounds     : list<sound>
  next_ambient_time  : int (ms)
```

The split between *controlled* entity and *viewed* entity is load-bearing: a cutscene, a death camera or a spectator mode views one entity while control stays with another, or with none.

## `load`

**Contract** — Loads one level by index. Blocking, several seconds, and it drives the loading screen through named progress stages. Every step depends on the one before it. Fatal if the level's configuration file is missing or if the level's binary version does not match the one this build was compiled for — a mismatched level is refused rather than guessed at.

```text
FUNCTION load(level_index)
  persistent.set_level(level_index)                 # mounts "$level$" at this level's directory
  FAIL IF the level's configuration file is absent
  config = read it

  show("opening stream") ; stream = open the level's main binary file
  read the header chunk ; FAIL IF its version is not this build's

  show("loading cform")
  object_space.load(build, serialize, deserialize, remap_materials)   # see Notes
  spatial_index.initialize(object_space.bounds)
  physics_spatial_index.initialize(object_space.bounds)

  sound_scene.set_occlusion_geometry(object_space.static_model, object_space.bounds)
  sound_scene.on_sound_emitted = register_sound_event

  renderer.load_level(stream)                       # geometry, visibility topology, lightmaps
  environment.load_modifiers()                      # this level's local weather overrides
  game.before_objects_loaded()
  objects.load()                                    # spawns every entity in the spawn file
  close stream
  ready = true

  IF NOT dedicated_server THEN capture input ; register as a render handler
  register as a frame handler
```

**Invariants** — The order is not negotiable. The collision database must exist before the spatial indices, because they are sized to its bounds. The sound scene's occlusion geometry must be the collision model, so it comes after. The renderer's level load reads from the same stream and must come after the header. Objects come last because spawning an entity immediately queries collision and registers in the spatial index.

**Notes** — Two spatial indices are built over the same bounds, one for general queries and one for physics. They are separate because physics queries run at a different rate and against a different object set, and sharing one index would make every physics step contend with the AI's sense queries.

**Notes** — The collision database load is handed four callbacks rather than being a plain read, because the database is *cached*: it is built from the level's triangle soup the first time and serialized, and deserialized on subsequent loads. The four callbacks are the game's hooks — assign per-triangle game material after a build, write the game's extra per-triangle data into the cache, read it back, and remap material identifiers when the cache was built against a different material table. The fourth exists because materials are named in configuration that the player may have modded between runs, so a cache's material indices cannot be trusted.

**Notes** — Registering as a render handler is skipped for the dedicated server but registering as a frame handler is not. The server simulates and does not draw.

**Notes** — The sound scene is given a callback rather than the engine polling it. Sound emission happens deep inside the audio layer, on its own schedule, and the perception event must be raised at that moment.

## `stop`

**Contract** — Tears the level down. Runs the object update six times with rendering suppressed, then unloads every object, releases input capture and lowers the ready flag.

**Notes** — Six update passes before the unload. Each pass processes the object list's deferred-destruction queue, and destroying an object can queue further destructions — an entity releasing its inventory, a vehicle releasing its occupants. Six is empirically enough to reach a fixed point for the shipped content; the original marks it as unexplained. A rebuild should loop until the destruction queue is empty and assert a bound, which is what six is standing in for.

## `on_frame`

**Contract** — The level's per-frame simulation step: deliver last frame's sound perception events, update every object, update the heads-up display, and occasionally start a random ambient sound. Asserts the level is ready.

```text
FUNCTION on_frame()
  dispatch_sound_events()
  objects.update(no_rendering = false)
  hud.on_frame()
  IF ambient sounds exist AND global_time > next_ambient_time THEN
    next_ambient_time = global_time + random 10..20 seconds
    position = camera_position + a random direction at a random distance of 30..100 metres
    play a random ambient sound there, at full volume, audible from 10 to 200 metres
```

**Notes** — The random ambient is positioned on a sphere around the camera rather than at a fixed place, so it sounds like it comes from the world without the world having to contain an emitter. The distance band, 30 to 100 metres, keeps it clearly outside the player's immediate space; the audible range starting at 10 means it never attenuates to nothing at the near end of that band. These are feel constants with no other source.

**Notes** — Sound perception events are dispatched at the *start* of the frame, meaning they were raised during the previous frame's audio update. A creature therefore reacts to a sound one frame after it was played. That one-frame lag is invisible and is what lets the audio layer raise events from its own thread.

## `on_render`

**Contract** — Asks the renderer to compute the frame — visibility, culling, light lists — and then to draw it. On a dedicated server, sleeps for a configurable few milliseconds instead, to keep the process from spinning a core.

**Notes** — Calculate and render are two calls, not one, because the calculation is what the parallel render contexts fan out from; separating them is the seam at which a rebuild would overlap visibility with submission.

## `set_entity` / `set_view_entity`

**Contract** — Change which entity is controlled, or which is viewed. The outgoing entity is told it has lost the role and the incoming one that it has gained it, in that order. Setting the controlled entity also sets the viewed one; setting the viewed one alone does not disturb control.

**Invariants** — The lose notification precedes the gain notification, so an entity handing over to itself sees a clean lose-then-gain rather than a gain into a state it has not left.

## `register_sound_event`

**Contract** — Called by the audio layer the moment a sound begins playing. Finds every entity in range that is registered as reacting to sound, computes how loudly each perceives it after distance attenuation and geometric occlusion, and queues a perception event for each. Drops everything while no level is fully loaded, and drops sounds whose emitting entity is already being destroyed.

```text
FUNCTION register_sound_event(sound, range)
  IF the world is not loaded THEN RETURN
  IF the emitting entity is being destroyed THEN detach it from the sound ; RETURN
  IF the sound is not actually playing THEN RETURN

  clamp range to 0.1 .. 500 metres
  params = the sound's live playback parameters
  position = params.position
  IF the sound is non-positional THEN position = the listener's position
  range = min(range, params.max_ai_distance)

  listeners = spatial query for entities marked "reacts to sound" in a box of that range
  FOR EACH listener
    IF it is being destroyed THEN CONTINUE
    dist = distance from the emission point to the listener
    IF dist > params.max_ai_distance THEN CONTINUE
    power = (1 - dist / params.max_ai_distance) * params.volume
    IF power is negligible THEN CONTINUE
    power = power * occlusion_between(listener, emission point)
    IF power is negligible THEN CONTINUE
    queue (listener, sound, power)
```

**Invariants** — The AI-audible range is a property of the *sound asset*, carried in its sidecar, and is never larger than what the caller requests. A silenced weapon and a rifle differ here and nowhere else.

**Notes** — Perceived power falls off *linearly* with distance, not with the inverse square the audio mixer uses. Two different attenuation models for the same sound is deliberate: the audible level is physics and the perceptibility is gameplay, and a linear falloff gives a predictable, authorable "creatures within this radius notice this".

**Notes** — Occlusion is queried from the sound scene, which has the collision model as its geometry, so a wall between the emitter and a creature genuinely reduces what the creature hears. This is the one place AI perception and audio share a computation.

**Notes** — A non-positional sound is relocated to the listener's position, because a two-dimensional sound has no world position and a creature needs one. The dispatch step does the same substitution with the camera position, which is the same point.

**Notes** — The query box is a cube of the range rather than a sphere, and the distance test afterwards is against the *sound's* maximum AI distance rather than the clamped range. The box is a broad phase; the test is the real one.

## `dispatch_sound_events`

**Contract** — Drains the queue, delivering each perception event to its listener with the emitting entity, the sound's game type, its game payload, the emission position and the perceived power. Drains from the back, and skips a sound that stopped playing between queueing and dispatch.

## `on_listener_destroyed`

**Contract** — Removes every queued event addressed to a listener that is being destroyed. Part of the engine-wide guarantee that a destroyed entity is unreferenced everywhere before its memory is released — and it matters here because the queue holds listener pointers across a frame boundary.

## `check_textures`

**Contract** — Implemented in [`IGame_Level_check_textures.cpp`](IGame_Level_check_textures.cpp.md).

## Level teardown

**Contract** — Releasing the level unloads the renderer's level data, destroys the camera manager, deregisters from the render and frame sequences, resets post-processing to neutral, restores the persistent sound scene as the default, and destroys the level's own sound scene. Reports final texture memory.

**Notes** — Post-processing is reset here, not by whoever set it. A screen effect running when a level unloads has no owner left to stop it, and would otherwise tint the main menu.

**Notes** — A command-line switch makes the renderer retain textures across the unload, so a level reload does not re-read them from disk. That is a development convenience and it trades a large resident set for load time.

## `ServerInfo`

**Contract** — A small fixed-capacity list of name/value/colour lines the game fills in for a server browser. At most fifteen entries; further additions are dropped silently. Each entry is stored pre-joined as `name = value`.

**Notes** — Fifteen is the number of lines the server-info panel displays, so the cap is the UI's and the data structure enforces it rather than the UI truncating. A rebuild should keep the cap where the display is.
