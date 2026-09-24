# src/xrEngine/IGame_Level.h

> Declares the level: the single global handle through which every subsystem reaches the loaded world, and the contract the game module must fill.

**Needs** — [`IInputReceiver.h`](IInputReceiver.h.md) · [`xr_object_list.h`](xr_object_list.h.md) · [`xrCDB/xr_area.h`](../xrCDB/xr_area.h.md) · [`xrSound/Sound.h`](../xrSound/Sound.h.md) · [`EngineAPI.h`](EngineAPI.h.md) · [`EventAPI.h`](EventAPI.h.md) · [`pure.h`](pure.h.md)
**Used by** — [`light_gi.cpp`](../Layers/xrRender/light_gi.cpp.md) · [`r__sector_detect.cpp`](../Layers/xrRender/r__sector_detect.cpp.md) · [`engine_impl.cpp`](../editors/xrWeatherEngine/engine_impl.cpp.md) · [`CameraBase.cpp`](CameraBase.cpp.md) · [`Environment.cpp`](Environment.cpp.md) · [`Environment_editor.cpp`](Environment_editor.cpp.md) · [`Environment_misc.cpp`](Environment_misc.cpp.md) · [`Environment_render.cpp`](Environment_render.cpp.md) · [`FDemoPlay.cpp`](FDemoPlay.cpp.md) · [`FDemoRecord.cpp`](FDemoRecord.cpp.md) · [`Feel_Touch.cpp`](Feel_Touch.cpp.md) · [`Feel_Vision.cpp`](Feel_Vision.cpp.md) · [`IGame_Level.cpp`](IGame_Level.cpp.md) · [`IGame_Level_check_textures.cpp`](IGame_Level_check_textures.cpp.md) · _and 16 more_
**Tier floor** — T2: an interface and a handful of owned subsystems.

## Purpose

Declares the surface implemented in [`IGame_Level.cpp`](IGame_Level.cpp.md), plus everything the game module must supply. It is simultaneously an input receiver, a render handler, a frame handler and an event receiver — four roles, because the level is the thing the frame loop drives.

One process-wide reference points at the loaded level, or at nothing. Code throughout the engine and the game tests that reference for presence as "is a world loaded"; it is the engine's most-read global.

## Exported units

- **`Level`** — owns the object list, the collision area, the camera manager, the sound scene, the level's own configuration and the heads-up display. See the implementation twin.
- **`load` / `stop` / `on_frame` / `on_render`** — the lifecycle and the per-frame steps.
- **`name`** — the level's identifier, used to resolve per-level data paths. Game-supplied.
- **`net_start` / `net_load` / `net_save` / `net_stop` / `net_update`** — the session lifecycle. Starting takes a server description and a client description as two strings, which is how single-player, listen-server and dedicated-server modes are all expressed as one call. Saving and loading name a save slot. Game-supplied.
- **The four collision-database hooks** — assign game materials after a build; serialize and deserialize the game's own per-triangle data into the collision cache; remap material identifiers when the cache predates a configuration change. Game-supplied, and mandatory — the collision cache cannot be written or trusted without them.
- **`before_objects_loaded` / `after_objects_loaded`** — two points for the game to do its own level setup. The second is registered but not called; see the implementation.
- **`current_entity` / `current_view_entity` / `set_entity` / `set_view_entity`** — who is controlled and whose eyes are used.
- **`register_sound_event` / `dispatch_sound_events` / `on_listener_destroyed`** — the sound-perception pipeline.
- **`world_rendered`** — a per-frame flag saying whether the scene has been drawn yet, read by anything that must draw before or after it.
- **The environment time accessors** — get and set the weather clock and its time factor. Game-supplied, because game time is the game's: it advances with the simulation, survives a save, and accelerates when the player sleeps.
- **`open_demo_file` / `start_demo_playback`** — recorded-session playback. Game-supplied.
- **`ServerInfo`** — the bounded name/value list shown in a server browser.
- **`check_textures`** — texture budget reporting; see [`IGame_Level_check_textures.cpp`](IGame_Level_check_textures.cpp.md).

**Notes** — The engine declares this interface and the game implements it, while the game also links back against the engine. That is one of the three deliberate cycles the build order names, and the edge is broken here: the engine only ever holds the interface.
