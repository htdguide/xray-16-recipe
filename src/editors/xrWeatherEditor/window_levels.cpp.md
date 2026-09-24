# src/editors/xrWeatherEditor/window_levels.cpp

> One handler: when the level panel loses focus, the world view keeps editing its grid.

**Needs** — [`window_levels.h`](window_levels.h.md) · [`window_ide.h`](window_ide.h.md) · [`window_view.h`](window_view.h.md)
**Used by** — [`window_levels.h`](window_levels.h.md)
**Tier floor** — T3: one event handler.

## Purpose

The panel's entire behaviour, and it is one rule shared by every grid-bearing panel in the
tool.

## `on_losing_focus`

```text
ON focus_lost
  view.property_grid = this panel's grid
```

**Contract** — hands this grid to the three-dimensional view, which uses it as the target
for drag-to-increment and sample-a-colour. See
[`window_view.cpp`](window_view.cpp.md).

**Notes** — **Losing focus, not gaining it**, and that is the whole trick: the author
selects a row in a grid, then moves the pointer over the world — which takes focus away
from the grid. If the view latched on focus *gain*, the selection would be handed over and
then immediately replaced by the view itself. Latching on the way out means the view
remembers the last grid the author was working in, which is exactly what they meant.

The weather panel does the same; the weather-editor panel, which has three grids, latches
on the way in as well, because it must distinguish between them. See
[`window_weather_editor.cpp`](window_weather_editor.cpp.md).
