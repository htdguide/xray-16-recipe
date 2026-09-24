# src/editors/xrWeatherEngine/editor_environment_sound_channels_source.hpp

> Declares one sound file in a channel's pool: a browsable path and nothing else.

**Needs** — [`editor_environment_sound_channels_source.cpp`](editor_environment_sound_channels_source.cpp.md) · [`Include/editor/property_holder_base.hpp`](../../Include/editor/property_holder_base.hpp.md) · [`Common/Noncopyable.hpp`](../../Common/Noncopyable.hpp.md)
**Used by** — [`editor_environment_sound_channels_channel.cpp`](editor_environment_sound_channels_channel.cpp.md) · [`editor_environment_sound_channels_source.cpp`](editor_environment_sound_channels_source.cpp.md)
**Tier floor** — T3: a path and a file browser.

## Purpose

Declares the surface implemented in
[`editor_environment_sound_channels_source.cpp`](editor_environment_sound_channels_source.cpp.md).

## State

```text
RECORD Source
  path            : text            # a sound file, extension dropped
  property_holder : PropertyHolder
```

## Exported units

- **`fill`** — the one grid row: a browsable sound path.
- **`id`** — the path.
