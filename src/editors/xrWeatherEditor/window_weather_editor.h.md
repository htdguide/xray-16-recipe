# src/editors/xrWeatherEditor/window_weather_editor.h

> Declares the timeline panel: two scrubbers, a clock, a cycle and keyframe selector, and the three property grids that show a moment in the day.

**Needs** — [`window_weather_editor.cpp`](window_weather_editor.cpp.md) · [`window_ide.h`](window_ide.h.md) · [`ide_impl.hpp`](ide_impl.hpp.md) · [`Include/editor/engine.hpp`](../../Include/editor/engine.hpp.md) · [`Include/editor/ide.hpp`](../../Include/editor/ide.hpp.md)
**Used by** — [`ide_impl.cpp`](ide_impl.cpp.md) · [`window_ide.cpp`](window_ide.cpp.md) · [`window_ide.h`](window_ide.h.md) · [`window_ide_serialize.cpp`](window_ide_serialize.cpp.md) · [`window_weather.cpp`](window_weather.cpp.md) · [`window_weather_editor.cpp`](window_weather_editor.cpp.md)
**Tier floor** — T2: it holds a native engine reference and four bound callables into it.

## Purpose

Declares the surface implemented in
[`window_weather_editor.cpp`](window_weather_editor.cpp.md). This is the tool's centre:
everything an author does to a moment in the day happens here.

## State

```text
RECORD WeatherEditorPanel
  # selection
  cycle_combo        : ComboBox        # which weather cycle
  frame_combo        : ComboBox        # which keyframe within it
  previous, next     : Button          # step the keyframe selection, wrapping
  pause              : Button          # two-image toggle: 0 = running, 1 = paused
  time_factor        : NumericUpDown   # 0.1 .. 100000, step 1, one decimal place

  # the two scrubbers, each an integer 0..1000
  day_track          : TrackBar        # position across the whole day
  frame_track        : TrackBar        # position within the current blend

  clock              : MaskedTextBox   # "00:00:00" — the current time of day

  # the three grids
  current            : PropertyGrid    # the keyframe the blend starts from
  blend              : PropertyGrid    # the interpolated result, right now
  target             : PropertyGrid    # the keyframe the blend runs toward

  # commands
  copy               : Button          # the blend, to the clipboard
  paste_current      : Button          # clipboard into the start keyframe
  paste_target       : Button          # clipboard into the target keyframe
  create_from        : Button          # the blend, as a new keyframe
  reload_current     : Button          # revert the start keyframe
  reload_target      : Button          # revert the target keyframe

  # the four data sources, bound from the engine side
  cycle_names, cycle_count, frame_names_of, frame_count_of

  # re-entry guards, one per two-way control
  updating_frame_combo, updating_frame_track,
  updating_clock, updating_day_track : bool
  updates_enabled    : bool            # false while a combo box is dropped down
  load_finished      : bool
  pointer_down       : bool            # a scrubber is being dragged
```

**Invariants** — the three grids always show, in order, the keyframe the blend starts
from, the interpolated result, and the keyframe it runs toward. Each two-way control has a
guard that is set while the code writes it and tested when the control reports a change.

## Layout

Two scrubber rows at the top and bottom of a header band, with the cycle and keyframe
selectors, the step buttons, the pause toggle, the time factor and the clock between them;
below, the three grids side by side, separated by splitters, each with its own small
toolbar. The two outer grids are each given forty per cent of the panel's width when it
resizes. Labelled `weather editor`.

## Exported units

- **`bind_weather_editor`** — install the four data sources and populate the selectors.
- **`on_load_finished`** — attach the blend grid and restore the saved state.
- **`on_idle`** — the per-frame refresh.
- **`refill`** — rebuild the selectors after a reload.
- **`save` / `load`** — the three grids' persisted state.
