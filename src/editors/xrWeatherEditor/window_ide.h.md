# src/editors/xrWeatherEditor/window_ide.h

> Declares the application's main window: the frame that owns the dock, the four panels and the engine reference.

**Needs** — [`window_ide.cpp`](window_ide.cpp.md) · [`window_ide_serialize.cpp`](window_ide_serialize.cpp.md) · [`Include/editor/engine.hpp`](../../Include/editor/engine.hpp.md) · [`Include/editor/ide.hpp`](../../Include/editor/ide.hpp.md) · [`window_view.h`](window_view.h.md) · [`window_levels.h`](window_levels.h.md) · [`window_weather.h`](window_weather.h.md) · [`window_weather_editor.h`](window_weather_editor.h.md)
**Used by** — [`entry_point.cpp`](entry_point.cpp.md) · [`ide_impl.cpp`](ide_impl.cpp.md) · [`ide_impl.hpp`](ide_impl.hpp.md) · [`property_collection_editor.cpp`](property_collection_editor.cpp.md) · [`window_ide.cpp`](window_ide.cpp.md) · [`window_ide_serialize.cpp`](window_ide_serialize.cpp.md) · [`window_levels.cpp`](window_levels.cpp.md) · [`window_levels.h`](window_levels.h.md) · [`window_view.cpp`](window_view.cpp.md) · [`window_view.h`](window_view.h.md) · [`window_weather.cpp`](window_weather.cpp.md) · [`window_weather.h`](window_weather.h.md) · [`window_weather_editor.cpp`](window_weather_editor.cpp.md) · [`window_weather_editor.h`](window_weather_editor.h.md)
**Tier floor** — T2: a top-level window holding a reference to a native engine object.

## Purpose

Declares the surface implemented in [`window_ide.cpp`](window_ide.cpp.md) and
[`window_ide_serialize.cpp`](window_ide_serialize.cpp.md).

## State

```text
RECORD MainWindow
  dock            : DockPanel              # fills the frame; hosts every panel
  view            : ViewPanel              # the engine's rendering surface
  levels          : LevelsPanel            # level-to-cycle assignment
  weather         : WeatherPanel           # the weather model tree
  weather_editor  : WeatherEditorPanel     # the timeline and the three keyframe grids
  engine          : EngineHost             # the native engine, borrowed
  editor          : Ide                    # this application's own interface, borrowed
  window_rectangle : (left, top, width, height)   # the restored size, tracked live
```

**Invariants** — `window_rectangle` tracks the window's geometry **only while it is
neither maximised nor minimised**, so the saved position is the one to restore to.

## Layout

One dock filling the frame, in single-document mode, with the view as the document and
the other three docked to its right. The initial frame is 800 by 600 and the window opens
maximised on a first run.

## Exported units

- **construct / destruct** — build the panels, restore the session, tear down.
- **`view` / `levels` / `weather` / `weather_editor`** — the four panels.
- **`engine` / `ide`** — the two borrowed interfaces.
- **`base_registry_key`** — where the session is stored. See
  [`window_ide_serialize.cpp`](window_ide_serialize.cpp.md).
- **window events** — size, position, activation, closing.
