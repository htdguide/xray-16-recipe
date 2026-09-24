# src/editors/xrWeatherEngine/editor_environment_weathers_manager.hpp

> Declares the owner of every weather cycle: the set of files this editor exists to author.

**Needs** — [`editor_environment_weathers_manager.cpp`](editor_environment_weathers_manager.cpp.md) · [`property_collection_forward.hpp`](property_collection_forward.hpp.md) · [`Common/Noncopyable.hpp`](../../Common/Noncopyable.hpp.md) · [`Include/editor/property_holder_base.hpp`](../../Include/editor/property_holder_base.hpp.md)
**Used by** — [`editor_environment_levels_manager.cpp`](editor_environment_levels_manager.cpp.md) · [`editor_environment_manager.cpp`](editor_environment_manager.cpp.md) · [`editor_environment_weathers_manager.cpp`](editor_environment_weathers_manager.cpp.md) · [`editor_environment_weathers_weather.cpp`](editor_environment_weathers_weather.cpp.md) · [`engine_impl.cpp`](engine_impl.cpp.md)
**Tier floor** — T2: it owns the cycles and mediates every edit to them.

## Purpose

Declares the surface implemented in
[`editor_environment_weathers_manager.cpp`](editor_environment_weathers_manager.cpp.md):
the list of weather cycles, the four reload granularities, and the clipboard operations on
a single keyframe.

## State

```text
RECORD WeathersManager
  cycles        : list<Weather>       # one per file in the weather folder
  collection    : PropertyCollection  # makes that list editable in the grid
  cycle_ids     : list<text>          # cached, sorted; rebuilt when changed is set
  frame_ids     : list<text>          # scratch, rebuilt on every request — see the note
  changed       : bool                # set by the collection on any add/remove/rename
  environment   : EditorEnvironment
```

**Invariants** — `cycle_ids` is valid exactly while `changed` is clear. `frame_ids` has no
validity flag at all, which is the defect documented in the implementation.

## Exported units

- **`load` / `save` / `reload`** — read every cycle file, write them all, or discard and
  re-read.
- **`fill`** — add the cycle list to the grid and install the timeline's four data sources.
- **`cycle_ids`** — the sorted cycle names, for every picker that references a cycle.
- **`unique_id`** — the renaming rule for cycles.
- **`save_current_blend`** — serialise the interpolated keyframe to a text buffer.
- **`paste_current_time_frame` / `paste_target_time_frame` / `add_time_frame`** — the three
  ways a serialised keyframe can come back in.
- **`reload_current_time_frame` / `reload_target_time_frame` / `reload_current_weather`** —
  the finer reload granularities.
