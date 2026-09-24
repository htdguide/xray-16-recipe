# src/editors/xrWeatherEngine/editor_environment_ambients_sound_id.hpp

> Declares one entry in an ambient's sound-channel list: a name chosen from the channels the model defines.

**Needs** — [`editor_environment_ambients_sound_id.cpp`](editor_environment_ambients_sound_id.cpp.md) · [`Include/editor/property_holder_base.hpp`](../../Include/editor/property_holder_base.hpp.md) · [`Common/Noncopyable.hpp`](../../Common/Noncopyable.hpp.md)
**Used by** — [`editor_environment_ambients_ambient.cpp`](editor_environment_ambients_ambient.cpp.md) · [`editor_environment_ambients_sound_id.cpp`](editor_environment_ambients_sound_id.cpp.md)
**Tier floor** — T3: a name and a picker.

## Purpose

Declares the surface implemented in
[`editor_environment_ambients_sound_id.cpp`](editor_environment_ambients_sound_id.cpp.md).
The sound-channel counterpart of
[`effect_id`](editor_environment_ambients_effect_id.hpp.md), identical in shape.

## State

```text
RECORD SoundId
  id             : text                     # names a record in the sound channels file
  sounds_manager : SoundChannelsManager     # read-only; the source of the picker's options
  property_holder : PropertyHolder
```

## Exported units

- **`fill`** — the one grid row: a name chosen from a list.
- **`id`** — the chosen name.
