# src/editors/xrWeatherEditor/window_weather.h

> Declares the weather-model panel: a property grid under a three-button toolbar — save, revert this cycle, revert everything.

**Needs** — [`window_weather.cpp`](window_weather.cpp.md) · [`window_ide.h`](window_ide.h.md)
**Used by** — [`ide_impl.cpp`](ide_impl.cpp.md) · [`window_ide.cpp`](window_ide.cpp.md) · [`window_ide.h`](window_ide.h.md) · [`window_ide_serialize.cpp`](window_ide_serialize.cpp.md) · [`window_weather.cpp`](window_weather.cpp.md)
**Tier floor** — T3: a dockable panel holding a grid and a toolbar.

## Purpose

Declares the surface implemented in [`window_weather.cpp`](window_weather.cpp.md). This is
where the weather model is authored: the tree built by
[`editor_environment_manager.cpp`](../xrWeatherEngine/editor_environment_manager.cpp.md)
appears in this grid.

## State

```text
RECORD WeatherPanel
  toolbar       : ToolStrip
  save          : Button      # "save weathers"
  reload_cycle  : Button      # "reload current weather only"
  reload_all    : Button      # "reload all the weathers"
  property_grid : PropertyGrid
  main_window   : MainWindow  # borrowed
```

## Layout

A toolbar across the top with three image buttons, a grid filling the rest with its own
toolbar hidden. Labelled `weather`.

## Exported units

- **`property_grid`** — the grid, so the engine side can attach its content to it.
- **the three commands** — save, revert this cycle, revert everything.
- **on losing focus** — hand this grid to the view as the drag-and-sample target.
