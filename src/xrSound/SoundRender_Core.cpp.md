# src/xrSound/SoundRender_Core.cpp

> The target-agnostic half of the sound manager: it owns the source cache, the scene list, the
> two clocks every emitter reads, and the tunable constants the whole chapter is calibrated around.

**Needs** — [`SoundRender_Core.h`](SoundRender_Core.h.md) · [`SoundRender_Source.h`](SoundRender_Source.h.md) · [`SoundRender_Emitter.h`](SoundRender_Emitter.h.md) · [`SoundRender_Scene.h`](SoundRender_Scene.h.md) · [`Sound.h`](Sound.h.md) · [`Include/xrAPI/xrAPI.h`](../Include/xrAPI/xrAPI.h.md) · [Seam: Audio device](../../SYSTEM-REQUIREMENTS.md#seam-audio-device)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: no device calls and no byte layouts here; it only fails T3 because it sits inside a fixed per-frame budget shared with the renderer.

## Purpose

The sound manager is split in three: an abstract interface the rest of the engine sees
([`Sound.h`](Sound.h.md)), this target-agnostic core, and a device-specific subclass
([`SoundRender_CoreA.cpp`](SoundRender_CoreA.cpp.md)). This file is the middle layer — everything
that would be identical no matter which mixer is underneath. The split exists because exactly four
operations are device-specific (open device, create voices, set listener transform, set master
gain); everything else — caching decoded sources, ranking emitters, blending reverb presets — is
policy, and policy is what a rebuild must reproduce.

The file also declares the module's tunables. They are console variables in the shipped engine, so
treat the values below as *defaults a player can change*, not as constants.

## State

```text
RECORD SoundCore
  scenes        : list<Scene>        # usually one; the editor opens several
  sources       : map<text, Source>  # key: asset path, lowercased, extension stripped
  sources_lock  : mutex              # sources is read from the streaming worker
  targets       : list<Target>       # the hardware voices, allocated once at startup
  emitter_epoch : int                # bumped once per update; see the Processor twin
  listener      : Listener
  listener_moved: bool
  env_current   : Environment        # what the device is hearing right now
  env_target    : Environment        # what the listener's region says it should hear
  effects       : optional<Effects>  # none when the device has no reverb
  clock         : Clock              # scaled by the game's time factor
  clock_steady  : Clock              # never scaled
  clock_value, clock_delta               : real
  clock_steady_value, clock_steady_delta : real
  present       : bool               # a device exists at all
  ready         : bool               # the device is initialized and may be driven
  locked        : bool               # an update or render pass is in progress
  supports_float_pcm : bool

RECORD Listener
  position    : vector
  orientation : vector[3]   # forward, up, right
```

Invariants:

- `sources` is a cache that is never evicted while the engine runs. A source is shared by every
  emitter playing that asset; releasing a reference is a no-op (see
  [`SoundRender_Core_SourceManager.cpp`](SoundRender_Core_SourceManager.cpp.md)).
- `targets` is fixed in size from initialization to shutdown. Voices are never created on demand
  — the whole ranking machinery exists because this list cannot grow.
- `locked` is true exactly while the engine is inside update or render. Every entry point the game
  calls asserts it is false, which makes "do not play a sound from a sound callback" a checked
  rule rather than a convention.
- The engine keeps two clocks because a sound may opt out of the game's time-scaling: a slow-motion
  effect must not stretch the user-interface click.

## Tunables

```text
targets_max        = 32     # hardware voices requested; the device may give fewer
occlusion_scale    = 0.5    # volume multiplier for a source behind level geometry
time_factor        = 1.0    # game speed; scales both pitch and the emitter clock
cull_volume        = 0.01   # below this smoothed volume an emitter is not worth a voice
rolloff            = 0.75   # distance-attenuation exponent handed to the device
volume_effects     = 1.0    # player's effects slider
volume_factor      = 1.0    # engine-side master trim for effects
volume_music       = 1.0    # player's music slider
cache_size_mb      = 32
flags              = { hardware_mixing, reverb, float32_pcm }
```

`cull_volume = 0.01` is the hinge of the whole chapter: it is roughly −40 dB, quiet enough to be
inaudible under any other sound, and it is compared against a *smoothed* volume so that a source
hovering at the threshold does not flicker. `targets_max = 32` matches the floor the audio seam
promises; asking for more and accepting fewer is the documented behaviour.

## `construct`

**Contract** — Binds the core to its owning manager and puts both environments at identity (a
reverb-free room), both clocks at their current reading, and the epoch at zero. Nothing touches the
device: a core exists before a device is known to exist, because the device list must be
enumerable by the settings screen before a device is opened.

## `initialize` / `clear`

**Contract** — `initialize` starts both clocks and marks the core present and ready; the subclass
calls it after the device and its voices exist. `clear` marks the core not-ready and destroys every
cached source. Neither blocks. Order matters: the subclass destroys voices *around* this call, so
that no voice is still reading a source's decoder when the source dies.

## `create_scene` / `destroy_scene`

**Contract** — A scene is an independent world of emitters and geometry with its own occlusion and
environment databases; the core only holds the list so that the update pass can walk them all. The
game runs one, the level editor runs one per open window. Destroying a scene removes it from the
list first and then deletes it, so a scene's destructor (which stops its emitters) never races the
update walk.

**Notes** — A rebuild may collapse this to a single scene if it does not ship an editor; the cost
is that the sound module then cannot be reused by tools.

## `create`

**Contract** — Turns an asset path into a playable sound handle. Strips any extension the caller
supplied (callers are inconsistent about writing `.ogg`), resolves or loads the shared source, and
wraps it in a handle carrying the per-instance facts: total bytes, total seconds, the sound type
(effect or music, which selects the volume slider), and the AI game type. Returns nothing when no
device is present or the asset is missing — a missing sound is not fatal, the game plays silence.

```text
FUNCTION create(path, sound_type, game_type) -> optional<Sound>
  IF NOT present THEN RETURN none
  key ← strip_extension(path)
  source ← get_or_load_source(key)
  IF source IS none THEN RETURN none
  snd ← Sound{ handle: source, s_type: sound_type }
  # sg_SourceType asks the asset to speak for itself: the sidecar's own
  # game type wins over whatever the caller guessed.
  snd.g_type ← IF game_type = FROM_SOURCE THEN source.game_type ELSE game_type
  snd.bytes_total ← source.bytes_total
  snd.time_total  ← source.length_sec
  RETURN snd
```

## `attach_tail`

**Contract** — Appends a second (and at most a third) asset to a sound so that it plays as one
uninterrupted stream. Used for weapon fire, where a shot's body and its tail are separate assets
that must not have a seam between them. Extends the handle's byte and time totals immediately, and
if the sound is already playing, pushes out the emitter's stop time by the tail's length. Refuses
a third attachment and logs it outside shipping builds.

**Invariants** — At most two pending attachments. The emitter consumes them in order: when its
stream cursor passes the end of the current asset it swaps handles and shifts the queue down (see
[`SoundRender_Emitter.cpp`](SoundRender_Emitter.cpp.md)).

**Notes** — The tail source is loaded here purely to read its length, then "destroyed" — which is a
no-op, so the effect is to warm the cache before the emitter needs it mid-stream. That is the real
reason the call is made at attach time rather than at swap time: swapping happens on the streaming
worker, where a synchronous file load would starve the voice.

## `destroy`

**Contract** — Releases a sound handle. Stops its emitter immediately if it has one (not deferred —
the handle is going away, so there is no one left to fade), asserts the emitter cleared its
back-pointer, and drops the source reference. Called from the handle's own teardown, so it must
tolerate being reached while the handle is half-dead.

**Invariants** — After this returns, no emitter and no event queue may still name this handle. The
emitter's stop path is what guarantees the second half of that.

## `stop_emitters` / `pause_emitters`

**Contract** — Broadcast to every scene. Pause returns the resulting nesting depth, because pausing
is counted, not boolean: the menu, a cutscene and the console can each independently want silence,
and sound resumes only when the last of them releases. See
[`SoundRender_Scene.cpp`](SoundRender_Scene.cpp.md) for how the count is matched against emitters.

## `update_listener`

**Contract** — Records the listener's position and orientation, and advances the reverb blend. Runs
once per update, after every emitter has been updated.

```text
FUNCTION update_listener(position, forward, up, right, dt)
  IF position differs from listener.position THEN
    listener.position ← position
    listener_moved ← true
  listener.orientation ← (forward, up, right)

  IF reverb disabled OR effects IS none THEN RETURN

  IF listener_moved THEN
    listener_moved ← false
    env_target ← scene.environment_at(position)   # one ray cast, see Scene
  env_current ← lerp(env_current, env_target, clock_delta)
  effects.set_listener(env_current)
  effects.commit()
```

**Notes** — Two decisions hide in four lines. First, the region query runs *only when the listener
moved*, because it is a ray cast against the environment database and a stationary listener cannot
change rooms. Second, the blend factor is the frame delta itself, which makes this an exponential
approach whose time constant is one second only at 1 fps — it is frame-rate dependent, and a
rebuild that wants the original's feel should reproduce it rather than "fix" it to a proper
exponential, because level designers tuned boundary crossings against this behaviour.

The subclass overrides this to also push the listener transform at the device, converting from the
engine's left-handed world to the mixer's right-handed one.

## `env_apply`

**Contract** — Forces the reverb blend to re-evaluate on the next update by claiming the listener
moved. Called whenever the environment *library* changes underneath a stationary listener: a level
loads its environment geometry, the editor reloads the preset file, or a script overrides the
region.

## `refresh_sources`

**Contract** — Stops every emitter, then reloads every cached source from disk in place. Blocks.
This is a modding affordance: a modder edits a sound asset and refreshes without restarting. It is
safe only because stopping every emitter first guarantees no decoder is open on any source.

## `restart`

**Contract** — Re-applies the environment. The device-level restart (voices and context) is the
subclass's job; at this level a restart is only "the world may have changed, re-derive the reverb".
