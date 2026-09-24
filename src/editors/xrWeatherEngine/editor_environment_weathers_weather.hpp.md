# src/editors/xrWeatherEngine/editor_environment_weathers_weather.hpp

> Declares one weather cycle: a named, ordered set of keyframes that is also one file on disk.

**Needs** — [`editor_environment_weathers_weather.cpp`](editor_environment_weathers_weather.cpp.md) · [`property_collection_forward.hpp`](property_collection_forward.hpp.md) · [`Include/editor/property_holder_base.hpp`](../../Include/editor/property_holder_base.hpp.md) · [`Common/Noncopyable.hpp`](../../Common/Noncopyable.hpp.md)
**Used by** — [`editor_environment_weathers_manager.cpp`](editor_environment_weathers_manager.cpp.md) · [`editor_environment_weathers_time.cpp`](editor_environment_weathers_time.cpp.md) · [`editor_environment_weathers_weather.cpp`](editor_environment_weathers_weather.cpp.md)
**Tier floor** — T2: it owns its keyframes and its file.

## Purpose

Declares the surface implemented in
[`editor_environment_weathers_weather.cpp`](editor_environment_weathers_weather.cpp.md).

## State

```text
RECORD Weather
  id          : text                  # also the file name, without extension
  frames      : list<Time>            # sorted by time of day
  collection  : PropertyCollection
  property_holder : PropertyHolder
  environment : EditorEnvironment
```

**Invariants** — `frames` is sorted by keyframe identifier, which is a time of day in
`HH:MM:SS` form, so text order and time order coincide. The cycle's entry in the
environment's cycle table is kept in step with `frames` element for element.

## Exported units

- **`load` / `save` / `reload`** — read the cycle's file, write it, or discard and re-read.
- **`fill`** — the cycle's two grid rows: its name, and its keyframe list.
- **`id`** — the cycle's name.
- **`times`** — its keyframes, in order.
- **`unique_id` / `generate_unique_id`** — the keyframe naming rule, which is a clock, not
  a counter.
- **`save_time_frame` / `paste_time_frame` / `add_time_frame` / `reload_time_frame`** —
  the four single-keyframe operations.
