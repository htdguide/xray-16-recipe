# src/editors/xrWeatherEditor/ide_impl.cpp

> The editor's side of the engine contract: two window handles, the application loop, the per-frame repaint, and the factory for property holders.

**Needs** — [`ide_impl.hpp`](ide_impl.hpp.md) · [`Include/editor/ide.hpp`](../../Include/editor/ide.hpp.md) · [`Include/editor/engine.hpp`](../../Include/editor/engine.hpp.md) · [`window_ide.h`](window_ide.h.md) · [`window_view.h`](window_view.h.md) · [`window_levels.h`](window_levels.h.md) · [`window_weather.h`](window_weather.h.md) · [`window_weather_editor.h`](window_weather_editor.h.md) · [`property_holder.hpp`](property_holder.hpp.md) · [`property_container.hpp`](property_container.hpp.md) · [Seam: Windowing and input](../../../SYSTEM-REQUIREMENTS.md#seam-windowing-and-input)
**Used by** — [`ide_impl.hpp`](ide_impl.hpp.md)
**Tier floor** — T1: it hands the engine raw native window handles to create a graphics device against.

## Purpose

Everything the engine host may ask of the editor, implemented. Almost every method here is one line, and that is the finding: **the editor's obligations to the engine are thin**. The two halves are joined at a handful of points, and all of the editor's real content is in the windows this file delegates to.

## State

```text
RECORD Ide
  engine    : EngineFacade          # the host, never owned
  window    : optional<MainWindow>  # owned; assigned just after construction
  paused    : bool
  in_idle   : bool                  # invariant: true only between begin_idle and end_idle
```

## Window handles

**Contract** — `main_handle` returns the application's top-level window; `view_handle` returns the child surface inside the view panel that the engine creates its graphics device against.

**Invariants** — the two are different windows, and the view handle is the *drawing* child of the view panel rather than the panel itself. That is what keeps the engine's rendered output from covering the panel's own chrome.

**Notes** — the handle is read out of the managed window object as an integer of the platform's pointer width. Entirely incidental, and the reason the file has two spellings of the same three methods.

## `run`

**Contract** — enters the application's message loop with the main window as the loop's root, and does not return until the application quits. The engine's frames happen inside it through the idle handler installed in [`entry_point.cpp`](entry_point.cpp.md).

## The idle bracket

**Contract** — `begin_idle` and `end_idle` mark the pump's entry and exit and assert non-re-entrancy; `advance_one_frame` does the editor's own per-frame work, which is exactly two things: let the weather timeline advance its playhead, and let the view panel do whatever it does per frame.

```text
FUNCTION advance_one_frame()
  window.weather_editor.advance_one_frame()   # the timeline scrubber follows game time
  window.view.advance_one_frame()
```

**Notes** — only two panels have per-frame work. The property grids do not: they repaint on demand and read their values through live bindings, so a value the engine changed is simply read again on the next paint. That is the pull-not-push design paying off — there is no per-frame synchronization pass anywhere in this editor.

## `on_load_finished`, `pause`

**Contract** — `on_load_finished` tells the view and the timeline that a level finished loading, which is their cue to enable themselves; the interface is dead while a level loads. `pause` forwards a pause to the view panel only.

## `environment`

**Contract** — forwards the host's weather system to any editor code that needs to read it directly rather than through a property binding.

## `create_property_holder` / `destroy`

**Contract** — the factory the engine host calls whenever it wants to describe an object to the grid. Constructs an empty [`property_holder`](property_holder.hpp.md) bound to the host's engine facade, optionally as an element of a collection and optionally with a back-reference to its containing object. Destruction releases it and clears the caller's reference.

**Invariants** — allocation and release both happen here, on the editor library's side, which is the same ownership rule the library's entry points follow and for the same reason: the two halves are separately built and may not free each other's memory.

## `environment_levels` / `environment_weathers`

**Contract** — take a holder the engine has already filled and bind it to the level browser panel and the weather browser panel respectively, by making its container the panel's grid's selected object.

**Notes** — the holder arrives as the abstract interface and is narrowed to the concrete implementation, because only the concrete one owns the container the grid actually binds to. That narrowing is the seam's cost: the interface deliberately says nothing about containers, so the only implementor has to assert that it is itself. A rebuild in one language passes the container directly.

## `weather_editor_setup`

**Contract** — installs the four pull callables described in [`ide.hpp`](../../Include/editor/ide.hpp.md) into the timeline panel: all weather cycle names, their count, a named cycle's keyframe identifiers, and their count.

**Notes** — the timeline asks these four questions every time it repaints and never caches an answer. See [`ide.hpp`](../../Include/editor/ide.hpp.md) for why.
