# src/xrEngine/IGame_Persistent.h

> Declares the process-lifetime layer, the hooks the game module must fill, and the main-menu interface.

**Needs** — [`IGame_Persistent.cpp`](IGame_Persistent.cpp.md) · [`IGame_ObjectPool.h`](IGame_ObjectPool.h.md) · [`Environment.h`](Environment.h.md) · [`EngineAPI.h`](EngineAPI.h.md) · [`ILoadingScreen.h`](ILoadingScreen.h.md) · [`ShadersExternalData.h`](ShadersExternalData.h.md) · [`pure.h`](pure.h.md) · [`EventAPI.h`](EventAPI.h.md)
**Used by** — [`Blender_Recorder_StandartBinding.cpp`](../Layers/xrRender/Blender_Recorder_StandartBinding.cpp.md) · [`ModelPool.cpp`](../Layers/xrRender/ModelPool.cpp.md) · [`dxLensFlareRender.cpp`](../Layers/xrRender/dxLensFlareRender.cpp.md) · [`dxRainRender.cpp`](../Layers/xrRender/dxRainRender.cpp.md) · [`light.cpp`](../Layers/xrRender/light.cpp.md) · [`r__sector.cpp`](../Layers/xrRender/r__sector.cpp.md) · [`r__sector_traversal.cpp`](../Layers/xrRender/r__sector_traversal.cpp.md) · [`editor_environment_thunderbolts_manager.cpp`](../editors/xrWeatherEngine/editor_environment_thunderbolts_manager.cpp.md) · [`engine_impl.cpp`](../editors/xrWeatherEngine/engine_impl.cpp.md) · [`CameraManager.cpp`](CameraManager.cpp.md) · [`FDemoRecord.cpp`](FDemoRecord.cpp.md) · [`Feel_Vision.cpp`](Feel_Vision.cpp.md) · [`IGame_Level.cpp`](IGame_Level.cpp.md) · [`IGame_ObjectPool.cpp`](IGame_ObjectPool.cpp.md) · _and 23 more_
**Tier floor** — T2: a declaration plus two small interfaces.

## Purpose

Declares the surface implemented in [`IGame_Persistent.cpp`](IGame_Persistent.cpp.md), and
— more importantly — declares the **extension points the game module fills in**. The
engine instantiates whatever the game module returns and drives it through these; it never
knows the concrete type. This half of the file is a contract a rebuild must satisfy, so it
is written out here rather than left to the implementation twin.

One global instance exists and is reached by name from everywhere in the engine. That is
the service-locator pattern the system requirements call out; a rebuild should inject it.

## Exported units

The concrete surface, all contracted in
[`IGame_Persistent.cpp`](IGame_Persistent.cpp.md):

- `Level_Scan`, `Level_ID`, `Level_Set`, `GetArchiveHeader` — the level catalogue.
- `LoadBegin`, `LoadEnd`, `LoadTitle`, `LoadStage`, `LoadDraw`, `ShowLoadingScreen` — the
  load bracket and the loading screen.
- `Prefetch`, `destroy_particles`, `IsLoaded`, `GameType` — session-scope services.
- `IsMainMenuActive`, `MainMenuActiveOrLevelNotExist` — "is there a world behind this".
- `OnEvent`, `OnFrame`, `OnAppStart`, `OnAppEnd`, `OnAppActivate`, `OnAppDeactivate` — the
  sequences it joins.
- `DumpStatistics` — the particle counts on the statistics overlay.

Public data other modules reach for directly: the `Environment` (weather), the two spatial
databases (objects and physics), the object pool, the loading screen, the sound scene, the
main menu, the shared shader-constant block, and the parsed session parameters.

## What the game module must supply

**Contract** — the engine calls each of these on the instance the game module created. All
have defaults, so a game module that implements only the first two is a valid one.

```text
create_level()  -> Level         # build the concrete level. REQUIRED: returning none is fatal
destroy_level(level)             # release it. must leave the reference cleared

pre_start(options)               # about to start: end the old game type if it changed
start(options)                   # started: begin the new game type if it changed
disconnect()                     # level gone: drop what pointed into it
on_game_start() / on_game_end()  # game *type* changed, not level
update_game_type()               # same type, new session: re-read the rules

can_be_paused()      -> bool     # refuse a pause during a cutscene or a match
get_current_dof(out) / set_base_dof(v)     # the depth-of-field the renderer applies
on_render_ppui_query() -> bool             # does this game draw an in-world interface pass?
on_render_ppui_main() / on_render_ppui_pp()  # ...and where in the frame it goes
on_sector_changed(sector)        # the camera crossed into another visibility sector
on_assets_changed()              # the file namespace changed; re-resolve cached paths
dump_statistics(font, alert)     # add rows to the statistics overlay
```

**Notes** — `on_render_ppui_query` is a two-call protocol: the renderer asks whether the
game has a post-process interface pass at all, and only then calls the two that draw it.
The query exists so the renderer can skip allocating a render target for a pass that will
not happen. A rebuild can make it one nullable callback.

`can_be_paused` is a *veto*, consulted by the device before pausing the clocks. The pause is
attempted regardless; the game decides.

## `IMainMenu`

**Contract** — the interface through which the engine opens, closes and interrogates the
main menu without depending on the interface layer. Four calls, and the last one is the
subtle one.

```text
INTERFACE MainMenu
  activate(on)               # open or close
  is_active() -> bool
  can_skip_scene_rendering() -> bool   # the menu fully covers the world: skip drawing it
  destroy_internal(force)              # release the screens; force ignores any refusal
```

**Notes** — `can_skip_scene_rendering` is a performance contract, not a cosmetic one: an
opaque main menu over a loaded level lets the renderer skip the entire world pass, which is
the difference between a menu at hundreds of frames per second and a menu that keeps
rendering a level nobody can see. The menu answers, because only it knows whether the
screen it is showing is opaque.

`destroy_internal` takes a force flag because the menu may legitimately refuse to tear
itself down mid-transition, and shutdown may not accept the refusal.

## `params` — the session descriptor

**Contract** — four lowercased, positional, slash-separated fields parsed from one string:
the level or spawn name, the game type, the alife flag, and the new-or-load mode. Fewer
than four fields leaves the rest empty. The same string shape appears on the command line,
in the console `start` command, and inside a recorded match's header. **Frozen**: shipped
shortcuts and scripts build it.
