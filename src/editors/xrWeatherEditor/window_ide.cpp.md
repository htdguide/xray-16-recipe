# src/editors/xrWeatherEditor/window_ide.cpp

> The application frame: builds the four panels, tracks the geometry worth restoring, and turns a window close into an engine quit.

**Needs** — [`window_ide.h`](window_ide.h.md) · [`window_view.h`](window_view.h.md) · [`window_levels.h`](window_levels.h.md) · [`window_weather.h`](window_weather.h.md) · [`window_weather_editor.h`](window_weather_editor.h.md) · [`window_ide_serialize.cpp`](window_ide_serialize.cpp.md) · [`Include/editor/engine.hpp`](../../Include/editor/engine.hpp.md) · [`Include/editor/ide.hpp`](../../Include/editor/ide.hpp.md) · [Seam: Windowing and input](../../../SYSTEM-REQUIREMENTS.md#seam-windowing-and-input)
**Used by** — [`window_ide.h`](window_ide.h.md)
**Tier floor** — T2: a window frame that holds a native engine reference.

## Purpose

The application's outermost object. Three of its decisions matter to a rebuild: **what the
four panels are and which one the engine renders into**, **which window geometry is worth
remembering**, and **who is allowed to end the session.**

## State

See [`window_ide.h`](window_ide.h.md).

## Construction

```text
FUNCTION construct(engine : EngineHost)
  suspend layout
  self.engine = engine
  view           = new ViewPanel(self)
  levels         = new LevelsPanel(self)
  weather        = new WeatherPanel(self)
  weather_editor = new WeatherEditorPanel(self, engine)
  dock.theme = the application theme
  restore_session()                      # see window_ide_serialize.cpp
  resume layout
```

**Contract** — the panels are created before the session is restored, because restoring a
dock layout re-attaches *existing* panels by name rather than creating them.

**Notes** — Layout is suspended across the whole construction so the frame is laid out once
rather than after every panel. Incidental to the widget toolkit; what survives is that the
restore must not be able to observe a half-built panel set.

## The four panels

```text
view           # the engine renders into this one; it is the dock's document
levels         # one row per level: which weather cycle it plays
weather        # the whole weather model as a property tree
weather_editor # the timeline, the three keyframe grids, and the clipboard commands
```

**Notes** — The split is by *task*, not by data: the weather panel is where records are
authored, the weather-editor panel is where a moment in the day is inspected, and the
levels panel is where the result is assigned. The engine's surface is a panel like any
other, which is what lets an author dock it, float it or tab it beside the grids — and is
the reason the engine is given a *child* window rather than the frame; see
[`Include/editor/ide.hpp`](../../Include/editor/ide.hpp.md).

## Tracking the geometry worth restoring

```text
ON size_changed OR position_changed
  IF window_state IS maximized OR minimized
    RETURN                                  # do not record a derived geometry
  window_rectangle = (location, size)
```

**Contract** — the remembered rectangle is the *restored* geometry, never the maximised or
minimised one.

**Notes** — This is the small correct decision that makes session restore behave: a window
saved while maximised must reopen maximised **and** remember where it was before, so
un-maximising puts it back. Recording the maximised bounds would lose that. The window
state is saved separately; see
[`window_ide_serialize.cpp`](window_ide_serialize.cpp.md).

Position changes are also forwarded to the view panel, which has to re-clip the mouse when
the frame moves.

## Activation

```text
ON activated   : forward TO view.on_activated
ON deactivated : forward TO view.on_deactivated
```

**Notes** — The frame forwards activation to the view because the engine's input capture
and cursor clipping are the view's business, and a docked panel does not receive frame-level
activation itself.

## Closing

```text
ON closing(event)
  event.cancel = true          # refuse the close
  save_session()
  engine.disconnect()          # ask the engine to quit instead
```

**Contract** — the window refuses to close. The session is written, and the engine is asked
to end; the engine then reports a quit request, which the application's own loop observes
and acts on.

**Notes** — **The engine, not the window, ends the session**, and this inversion is the
counterpart of the one in [`Include/editor/ide.hpp`](../../Include/editor/ide.hpp.md): the
editor owns the loop but not the decision to leave it. It is what makes a console `quit`
typed into the three-dimensional view behave exactly like closing the window — one exit
path, whichever way it is reached.

The session is written here rather than on the way out, because by the time the engine
finishes quitting the panels may already be gone.

## Teardown

The four panels are released explicitly, in creation order. The engine and the editor
interface are borrowed and are not released.
