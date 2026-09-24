# src/editors/xrWeatherEngine/editor_environment_sound_channels_channel.hpp

> Declares one sound channel: a pool of sound files, a distance range, and the four numbers that pace how often one plays.

**Needs** — [`editor_environment_sound_channels_channel.cpp`](editor_environment_sound_channels_channel.cpp.md) · [`xrEngine/Environment.h`](../../xrEngine/Environment.h.md) · [`property_collection_forward.hpp`](property_collection_forward.hpp.md) · [`Include/editor/property_holder_base.hpp`](../../Include/editor/property_holder_base.hpp.md) · [`Common/Noncopyable.hpp`](../../Common/Noncopyable.hpp.md)
**Used by** — [`editor_environment_ambients_ambient.cpp`](editor_environment_ambients_ambient.cpp.md) · [`editor_environment_sound_channels_channel.cpp`](editor_environment_sound_channels_channel.cpp.md) · [`editor_environment_sound_channels_manager.cpp`](editor_environment_sound_channels_manager.cpp.md)
**Tier floor** — T2: it is the engine's sound-channel record, extended, and owns its source list.

## Purpose

Declares the surface implemented in
[`editor_environment_sound_channels_channel.cpp`](editor_environment_sound_channels_channel.cpp.md).

## State

```text
RECORD Channel EXTENDS RuntimeSoundChannel
  id          : text                # the configuration section name
  sources     : list<Source>        # the editable view of the sound file list
  collection  : PropertyCollection
  # inherited:
  distance    : (min : real, max : real)   # metres
  period      : (int, int, int, int)       # seconds; see the implementation
```

## Exported units

- **`load` / `save`** — read and write one configuration section.
- **`fill`** — the channel's eight grid rows.
- **`sounds`** — the engine's accessor, unchanged.
- **`id`** — the record's name.
