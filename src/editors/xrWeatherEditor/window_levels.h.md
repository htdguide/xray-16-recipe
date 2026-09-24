# src/editors/xrWeatherEditor/window_levels.h

> Declares the level-assignment panel: one property grid and nothing else.

**Needs** — [`window_levels.cpp`](window_levels.cpp.md) · [`window_ide.h`](window_ide.h.md)
**Used by** — [`ide_impl.cpp`](ide_impl.cpp.md) · [`window_ide.cpp`](window_ide.cpp.md) · [`window_ide.h`](window_ide.h.md) · [`window_ide_serialize.cpp`](window_ide_serialize.cpp.md) · [`window_levels.cpp`](window_levels.cpp.md)
**Tier floor** — T3: a dockable panel holding one widget.

## Purpose

Declares the surface implemented in [`window_levels.cpp`](window_levels.cpp.md). The
thinnest of the four panels: it is a frame around a property grid, filled from the engine
side by
[`editor_environment_levels_manager.cpp`](../xrWeatherEngine/editor_environment_levels_manager.cpp.md).

## State

```text
RECORD LevelsPanel
  property_grid : PropertyGrid       # fills the panel; toolbar hidden
  main_window   : MainWindow         # borrowed
```

## Layout

One grid filling the panel, with its own toolbar hidden. The panel may dock on any edge or
float, hides rather than closes, and is labelled `level weathers`.

## Exported units

- **`property_grid`** — the grid, so the engine side can attach its content to it.
- **on losing focus** — hand this grid to the view as the drag-and-sample target.
