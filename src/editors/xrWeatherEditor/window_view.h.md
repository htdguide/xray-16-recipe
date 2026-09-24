# src/editors/xrWeatherEditor/window_view.h

> Declares the panel the engine renders into: a toolbar with two toggles over a bare surface.

**Needs** — [`window_view.cpp`](window_view.cpp.md) · [`window_ide.h`](window_ide.h.md) · [`Include/editor/engine.hpp`](../../Include/editor/engine.hpp.md)
**Used by** — [`ide_impl.cpp`](ide_impl.cpp.md) · [`property_collection_editor.cpp`](property_collection_editor.cpp.md) · [`window_ide.cpp`](window_ide.cpp.md) · [`window_ide.h`](window_ide.h.md) · [`window_ide_serialize.cpp`](window_ide_serialize.cpp.md) · [`window_levels.cpp`](window_levels.cpp.md) · [`window_view.cpp`](window_view.cpp.md) · [`window_weather.cpp`](window_weather.cpp.md) · [`window_weather_editor.cpp`](window_weather_editor.cpp.md)
**Tier floor** — T1: it hands a native window handle to the graphics device.

## Purpose

Declares the surface implemented in [`window_view.cpp`](window_view.cpp.md).

## State

```text
RECORD ViewPanel
  surface          : Panel        # the child window the graphics device binds to
  edit_toggle      : ToolbarButton  # checked = editing, unchecked = flying the camera
  pause_toggle     : ToolbarButton  # enabled only while editing
  engine           : EngineHost   # borrowed
  main_window      : MainWindow   # borrowed
  property_grid    : optional<PropertyGrid>  # whichever grid last had focus
  previous_pointer : (x, y)       # for drag-to-increment
  loaded           : bool         # whether the level has finished loading
```

**Invariants** — the pause toggle is enabled exactly while the edit toggle is checked. The
panel is the dock's document and cannot be closed.

## Layout

A toolbar across the top with the two toggles; below it a surface filling the rest. The
surface, not the panel, is what the graphics device is given.

## Exported units

- **`draw_handle`** — the surface's native handle, for the graphics device.
- **`on_load_finished` / `on_idle` / `pause`** — the three calls the application drives it
  with.
- **`property_grid`** — tell the view which grid the drag and sample gestures act on.
- **window events** — size, position, activation, paint, keys, pointer.
