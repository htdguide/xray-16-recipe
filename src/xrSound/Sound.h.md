# src/xrSound/Sound.h

> The whole of sound as the rest of the engine sees it: a handle you can play, a scene to play it
> in, a manager that owns the device, and the parameter record they trade.

**Needs** — [`xrCore/xr_resource.h`](../xrCore/xr_resource.h.md) · [`xrCore/_vector3d.h`](../xrCore/_vector3d.h.md) · [`xrCore/_flags.h`](../xrCore/_flags.h.md) · [`xrCore/xr_token.h`](../xrCore/xr_token.h.md) · [`xrCDB/xrCDB.h`](../xrCDB/xrCDB.h.md) · [Seam: Audio device](../../SYSTEM-REQUIREMENTS.md#seam-audio-device)
**Used by** — [`pch.hpp`](../editors/xrWeatherEditor/pch.hpp.md) · [`pch.hpp`](../editors/xrWeatherEngine/pch.hpp.md) · [`Engine.h`](../xrEngine/Engine.h.md) · [`Environment.h`](../xrEngine/Environment.h.md) · [`Feel_Sound.h`](../xrEngine/Feel_Sound.h.md) · [`IGame_Level.h`](../xrEngine/IGame_Level.h.md) · [`stdafx.h`](../xrEngine/stdafx.h.md) · [`HudSound.cpp`](../xrGame/HudSound.cpp.md) · [`HudSound.h`](../xrGame/HudSound.h.md) · [`Level_Bullet_Manager.h`](../xrGame/Level_Bullet_Manager.h.md) · [`GameMtlLib.h`](../xrMaterialSystem/GameMtlLib.h.md) · [`GameMtlLib_Engine.cpp`](../xrMaterialSystem/GameMtlLib_Engine.cpp.md) · [`Sound.cpp`](Sound.cpp.md) · [`SoundRender.h`](SoundRender.h.md) · _and 9 more_
**Tier floor** — T2: abstract interfaces and a small value record; nothing here is device- or
layout-facing.

## Purpose

This is an interface header with no implementation file, and it is the most widely included file in
the chapter: everything from weapons to the user interface to the alife simulation plays sounds
through it. It declares four things — a *handle*, a *scene*, a *manager*, and the parameter record
— and nothing about how any of them work. The rest of `xrSound` is one filling behind it.

The separation earns its keep twice: the game links against this without knowing a mixer exists, and
a build with sound disabled fills the same interface with nothing.

## State

```text
RECORD Sound                        # one playable instance; reference-counted
  source    : SourceHandle          # the shared asset description
  emitter   : optional<Emitter>     # set while playing; cleared by the emitter when it ends
  sound_type: ENUM { Effect, Music }# selects which volume slider applies
  game_type : int                   # the AI classification; see SoundRender_Source
  owner     : optional<GameObject>  # who is making this noise, for the AI
  user_data : optional<UserData>    # a script payload carried along with the sound
  attached  : text[2]               # up to two assets to play after this one, gaplessly
  bytes_total : int                 # of the whole concatenation
  time_total  : real

RECORD SoundParams
  position     : vector
  base_volume  : real    # authored in the asset's sidecar
  volume       : real    # this instance's trim, 0..1
  freq         : real    # playback rate; 1 is authored pitch
  min_distance : real    # full volume inside this radius
  max_distance : real    # inaudible beyond it
  max_ai_distance : real # how far it carries to a creature
```

Invariants:

- `emitter` and the emitter's own back-reference are set and cleared together. A handle whose sound
  has ended has no emitter, and every control operation on it becomes a silent no-op — which is
  what makes it safe for game code to hold a handle for a sound that finished long ago.
- `attached` fills front-first and is consumed front-first; at most two entries.
- Destroying the last reference to a handle stops its emitter and releases its source.

## Flags and vocabulary

```text
ENUM SoundType   { Effect, Music }          # which volume slider
FLAGS PlayFlags  { Looped, TwoD, IgnoreTimeFactor }
FLAGS DeviceFlags{ HardwareMixing, Reverb, Float32 }
GameTypeSentinel FROM_SOURCE                # "use the asset's own AI type"
```

`IgnoreTimeFactor` subscribes a sound to the unscaled clock: slow motion must not slow the
interface. `TwoD` means listener-relative — no distance, no occlusion, top of the voice ranking —
and is forced automatically on any stereo asset, which has no single point to be at.

## `SoundHandle`

**Contract** — A reference-counted handle with the control surface game code actually uses:
`create`, `destroy`, `clone`, `attach_tail`, `play`, `play_at_pos`, `play_no_feedback`, `stop`,
`stop_deferred`, `set_position`, `set_frequency`, `set_range`, `set_volume`, `set_priority`,
`set_time`, `get_params`, `set_params`, `get_length_sec`.

**Invariants** — Every control call is a no-op on a handle with no live emitter, and every one of
them asserts the sound system is not inside its update or render pass. That second rule is the
re-entry guard: playing or stopping a sound from within a sound callback would mutate the lists the
update pass is walking.

**Notes** — `stop` versus `stop_deferred` is the choice gameplay code makes most often and most
often wrongly. Deferred fades over about a tenth of a second and is what anything audible should
use; immediate is for teardown. See
[`SoundRender_Emitter_StartStop.cpp`](SoundRender_Emitter_StartStop.cpp.md).

`clone` copies the per-instance record and shares the asset, so two handles can play the same sound
independently. `play_no_feedback` goes further and gives the caller *no* way back to the sound —
fire-and-forget for one-shots nobody will ever move or stop.

The three `play` entry points route to a **default scene**, a module-wide mutable that the engine
fills at startup. That is the service-locator pattern the system requirements call out; a rebuild
should pass the scene explicitly, and the recipe notes here that what is being reached for is
always "the one world the player is in".

## `Scene` (interface)

**Contract** — What a sound world must offer. The full semantics of each operation are in
[`SoundRender_Scene.cpp`](SoundRender_Scene.cpp.md); what this interface *demands* is:

- three ways to start a sound, differing in whether the caller keeps a handle to it and whether a
  position is given up front;
- broadcast stop, and a *depth-returning* pause — pausing is nested, not boolean;
- an installable hearing handler, called once per queued sound event per update;
- three geometry installers: the reverb-region mesh, the sound-occlusion mesh, and the level's
  collision model (borrowed, not owned);
- a reverb-region query for a world point, plus a script override that replaces the whole region
  system with one preset;
- two occlusion queries — listener-to-point, and point-to-point with a caller-chosen dispersion;
- a notification that a game object is being destroyed, so emitters can drop it without stopping.

## `Manager` (interface)

**Contract** — What the engine's frame loop demands of sound. Creates and destroys scenes; restarts;
reports whether it is mid-pass; stops and pauses across every scene; sets the master volume; runs
the update and render passes with the listener's transform; reports statistics; refreshes sources
from disk.

**Invariants** — `update` precedes `render` in a frame, and `render` may be skipped entirely (a
dedicated server updates sound so that AI hearing works, and renders nothing). The `locked` flag is
true exactly across both, and every handle operation asserts against it.

The creation and destruction of sound handles are *protected* on this interface — only the handle
type may call them. That keeps handle lifetime in one place rather than spread across every caller.

## `SoundManager` (facade)

**Contract** — The thin object the engine actually holds: it builds the device list, creates and
destroys the renderer, loads and unloads the reverb preset library, and answers whether sound is
enabled at all. The preset library lives here — above any scene — because it is game data shared by
every level.

## `SoundStats` / `SoundStatsExt`

**Contract** — The debug reports. The compact one is three counters: voices actually sounding,
emitters simulated, hearing events raised. The extended one is a row per emitter for the in-game
sound debugger.

## Tunables exported here

The module publishes its console variables through this header so the console and the settings
screen can bind them: the volume sliders (effects, music, and an engine-side effects trim), the
distance rolloff, the occlusion scale, the game time factor, the voice count, the device flags, the
source cache size and the selected device index. Their defaults and meanings are in
[`SoundRender_Core.cpp`](SoundRender_Core.cpp.md).

## Notes

`SoundEnvironment` is declared here as an empty opaque type and filled by
[`SoundRender_Environment.h`](SoundRender_Environment.h.md). That is how the game passes reverb
presets around — through scripts, through level data, through the weather system — without any
knowledge of the reverb model. The opacity is the interface: a rebuild must let a preset be a value
the game can hold and hand back, and must not require the game to understand it.
