# src/editors/xrWeatherEditor/window_weather.cpp

> Three commands: write the model out, and the two coarse ways to throw away what has not been written.

**Needs** — [`window_weather.h`](window_weather.h.md) · [`window_ide.h`](window_ide.h.md) · [`window_view.h`](window_view.h.md) · [`window_weather_editor.h`](window_weather_editor.h.md) · [`Include/editor/engine.hpp`](../../Include/editor/engine.hpp.md)
**Used by** — [`window_weather.h`](window_weather.h.md)
**Tier floor** — T3: three command handlers.

## Purpose

The panel's whole behaviour. Two of the three commands are the editor's undo, at the two
coarsest of its four granularities; see
[`Include/editor/engine.hpp`](../../Include/editor/engine.hpp.md).

## The three commands

```text
ON save_clicked
  engine.save_weathers()

ON reload_cycle_clicked
  engine.reload_current_weather()
  weather_editor.refill()          # the keyframe list changed underneath the timeline

ON reload_all_clicked
  engine.reload_weathers()
  weather_editor.refill()

ON focus_lost
  view.property_grid = this panel's grid
```

**Contract** — save writes every authored file the editor owns. The two reloads discard
in-memory edits at cycle and whole-model scope, then rebuild the timeline's pickers,
because the keyframes they were listing no longer exist.

**Invariants** — a reload **must** be followed by rebuilding the timeline. The keyframe
records were destroyed; the timeline's combo boxes hold their names, and its selection
would address records that are gone.

**Notes** — There is no confirmation on either reload, and no dirty flag anywhere in the
tool, so an author who has spent an hour on a cycle can discard it with one button and no
warning. **A rebuild should track modification and confirm** — the model already has a
changed flag per editable list, for cache invalidation; the same flag answers this.

The refill after a reload is marked in the original as something that could be narrower —
only the current cycle's keyframes actually changed in the first case — and it could, but
the cost is four calls and a combo box rebuild. Not worth narrowing.

Saving and reverting live on the *model* panel rather than the timeline panel, which is
right: they act on the whole model, not on the moment being inspected. The timeline's own
narrower reverts are on its own toolbar.
