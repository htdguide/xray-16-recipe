# src/Include/editor/ide.hpp

> What the engine host may ask of the editor library: give me a window, run the application, and make me a property grid.

**Needs** — [`engine.hpp`](engine.hpp.md) · [`property_holder_base.hpp`](property_holder_base.hpp.md) · [`interfaces.hpp`](interfaces.hpp.md) · [Seam: Windowing and input](../../../SYSTEM-REQUIREMENTS.md#seam-windowing-and-input)
**Used by** — [`engine.hpp`](engine.hpp.md) · [`interfaces.hpp`](interfaces.hpp.md) · [`property_holder_base.hpp`](property_holder_base.hpp.md) · [`ide_impl.cpp`](../../editors/xrWeatherEditor/ide_impl.cpp.md) · [`ide_impl.hpp`](../../editors/xrWeatherEditor/ide_impl.hpp.md) · [`window_ide.cpp`](../../editors/xrWeatherEditor/window_ide.cpp.md) · [`window_ide.h`](../../editors/xrWeatherEditor/window_ide.h.md) · [`window_weather_editor.cpp`](../../editors/xrWeatherEditor/window_weather_editor.cpp.md) · [`window_weather_editor.h`](../../editors/xrWeatherEditor/window_weather_editor.h.md) · [`editor_environment_levels_manager.cpp`](../../editors/xrWeatherEngine/editor_environment_levels_manager.cpp.md) · [`editor_environment_manager_properties.cpp`](../../editors/xrWeatherEngine/editor_environment_manager_properties.cpp.md) · [`editor_environment_suns_flare.cpp`](../../editors/xrWeatherEngine/editor_environment_suns_flare.cpp.md) · [`editor_environment_suns_manager.cpp`](../../editors/xrWeatherEngine/editor_environment_suns_manager.cpp.md) · [`editor_environment_suns_sun.cpp`](../../editors/xrWeatherEngine/editor_environment_suns_sun.cpp.md) · _and 1 more_
**Tier floor** — T1: it hands back native window handles the engine binds a graphics device to.

## Purpose

The mirror of [`engine.hpp`](engine.hpp.md). That file is what the editor asks of the engine; this is what the engine asks of the editor. The editor owns the application: its windows, its message loop, its property grid widget, its menus. The engine needs three things from it — a window to render into, a loop to be pumped by, and a way to describe its data so the grid can show it.

The instance is created by the library's [initialize entry point](interfaces.hpp.md) and destroyed by its finalize.

## State

`Stateless` as an interface; the implementor is the editor application.

## `ide_base` — what the editor library must provide

### Windows and the loop

```text
FUNCTION main_window() -> WindowHandle       # the application's top-level window
FUNCTION view_window() -> WindowHandle       # the child the engine renders into
FUNCTION run()                               # enter the application's message loop; returns at quit
FUNCTION on_load_finished()                  # the level has finished loading
FUNCTION pause()                             # the engine asks the editor to stop pumping
FUNCTION environment() -> Environment        # the weather system, forwarded from the host
```

**Contract** — two window handles, not one, and the split is load-bearing: the graphics device is created against the *view* window, so the editor's docked panels, toolbars and menus are ordinary widgets outside the rendered surface and do not have to be composited by the engine. The main window is what dialogs parent themselves to.

`run` does not return until the application quits; the engine's frames happen inside it, via the idle handler that calls back into [`engine_base.on_idle`](engine.hpp.md). `on_load_finished` is the editor's cue to enable the interface, which is disabled while a level loads.

**Notes** — This is the inverted-control handshake stated completely: **the editor owns the loop and the windows; the engine owns a child surface and a frame callable.** A rebuild is free to invert it back — engine owns the loop, editor is a panel drawn by the debug overlay toolkit — which is what most modern engines do, and which would delete this whole file and [`engine.hpp`](engine.hpp.md) with it. That is a legitimate redesign and the recipe recommends it; what this pair documents is the *data* the two halves exchange, which survives either arrangement.

### Property holders

```text
FUNCTION create_property_holder(display_name : text,
                                collection : optional<PropertyCollection>,
                                owner : optional<PropertyOwner>) -> PropertyHolder
FUNCTION destroy(inout holder : PropertyHolder)
```

**Contract** — the engine host asks the editor for an empty property holder, fills it with its own fields (see [`property_holder_base.hpp`](property_holder_base.hpp.md)), and hands it to a grid. The optional collection makes the holder an element of an editable list — the grid then offers add, remove and reorder. The optional owner lets the grid navigate back from a nested holder to the object that contains it.

Destruction takes the caller's reference and clears it, the same out-parameter discipline as the library's entry points and for the same reason: allocation and release both belong to the editor library.

### Prepopulated browsers

```text
FUNCTION populate_level_list(holder : PropertyHolder)
FUNCTION populate_weather_list(holder : PropertyHolder)
```

**Contract** — two ready-made trees the editor builds itself, because their contents come from browsing the game data rather than from any engine object: the list of levels, and the list of weather cycles with their keyframes.

### Weather timeline data sources

```text
FUNCTION bind_weather_editor(weathers, weather_count, frames_of, frame_count_of)
```

**Contract** — installs four callables the editor's timeline widget pulls from whenever it redraws: all weather cycle names, how many there are, the keyframe identifiers of a named cycle, and how many that cycle has.

**Notes** — Pull, not push. The timeline never holds a copy of the weather structure; it asks every time it paints. That is why adding, renaming or deleting a keyframe needs no notification protocol — the next paint sees it. The cost is four calls per paint, which is nothing for a list of tens.

The callables are bound delegates over the host's own methods. A rebuild uses closures; what must survive is the *direction* — **the editor reads the weather structure on demand from the engine, and never mirrors it.** A mirrored copy is the classic source of editor/engine divergence and this design forecloses it.
